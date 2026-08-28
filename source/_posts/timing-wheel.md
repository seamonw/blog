---
title: 时间轮算法：原理、层次化实现和一组 Go Timer 接口
date: 2026-08-28 19:30:00
categories: 算法
tags:
	- 时间轮
	- golang
	- timer
	- kafka
---

定时器看起来是件小事：睡一会儿，到点做事。数量少的时候，标准库 `time.Timer` 完全够用。数量到十万、百万——连接空闲超时、心跳、重试退避、延迟消息——问题就变成另一件事：怎么让「加一个、取消一个」的代价不跟着定时器总数一起涨。

时间轮就是为这个场景准备的。本文先把单层轮和层次化轮讲清楚，再对照一份 Go 实现说明 `Timer` / `Ticker` 这组接口是怎么挂上去的。完整代码在 [seamonw/timingwheel](https://github.com/seamonw/timingwheel)。

## 堆定时器为什么会慢下来

Go 运行时的定时器是每个 P 上一棵四叉最小堆，按绝对到期时间排序。插入、删除都是 O(log n)。n 到百万时，log n 并不大，真正难受的是两件事：

- 每次插入都要上堆、下沉，缓存不友好
- 取消一个还没到期的定时器，同样要走一遍堆调整

连接超时这种任务有个特点：绝大多数定时器会被续期或取消，真正开火的是少数。你付的是「维护有序」的成本，用到的却经常只是「到点扫一眼」。

时间轮换了一种问法：不要全局有序，只要能回答「下一个 tick 该谁到期」。

## 单层时间轮

把时间切成等长的格子，格子排成一个环。环上有一根指针，每隔 `tick` 往前走一格，走到哪一格，就触发那一格上的全部任务。

```text
tick = 1s, wheelSize = 8，当前指向槽 0

  0   1   2   3   4   5   6   7
 [ ] [A] [ ] [B] [ ] [ ] [C] [ ]
  ^
  指针

A 还有 1s，B 还有 3s，C 还有 6s
```

插入一次延迟 `d` 的任务，落到槽 `(now/tick + d/tick) % wheelSize`。槽里用链表挂任务，插入、按节点删除都是 O(1)。指针往前走一格，把当前槽的链表整条摘下来执行。

这就是 Varghese 和 Lauck 在 1987 年那篇 *Hashed and Hierarchical Timing Wheels* 里的 hashed wheel：用哈希（取模）代替排序。

单层轮有两个硬伤。

**覆盖范围有限。** 8 个 1 秒槽只能表示 8 秒以内的延迟。要表示一天，要么把 `tick` 拉大（精度烂掉），要么把槽位加到 86400（内存和空转都难看）。

**空转。** 指针每个 tick 都要醒一次。轮子上只有几个任务时，你在为空气付 goroutine 唤醒。

层次化时间轮解决第一件，延迟队列解决第二件。

## 层次化：秒针、分针、时针

钟表自己就是多层时间轮。秒针转一圈，分针走一格；分针转一圈，时针走一格。每一层的一格，等于下一层转完一整圈。

```text
第一层  tick = 1ms,   20 槽，覆盖     20ms
第二层  tick = 20ms,  20 槽，覆盖    400ms
第三层  tick = 400ms, 20 槽，覆盖      8s
第 n 层 tick = 20^(n-1) ms
```

一个 80ms 后到期的任务进不了第一层（第一层只看 20ms 以内），被放到第二层。第二层的指针走到它那一格时，并不立刻执行，而是**降级**：按剩余时间重新插入更细的轮。剩余时间已经小于第一层的区间，就落到第一层的某个槽；到点再开火。

插入规则可以写成：

```text
if expiration < now + tick:
    已经到期，立刻跑
else if expiration < now + tick*wheelSize:
    放进本层对应槽
else:
    交给上一层（没有就现建）
```

上层不是多出来的定时器线程。每一层共享同一套「桶到期」通知，时钟推进时从细到粗一起走。上层的一格被推进，才触发那一格的降级。任务最终一定在最细那一层开火，精度就是第一层的 `tick`。

Kafka 用这套结构管请求的延时响应，Netty 的 `HashedWheelTimer` 是单层轮加较长的 tick。层次化的好处是：精度和覆盖范围不再互相拆台。

## 延迟队列：不要为空槽醒来

指针每个 tick 走一格，空槽也要醒。Kafka 的做法是给每个槽记一个绝对到期时间，把「下一桶何时到期」丢进一个最小堆。后台只睡到堆顶，到点再推进时钟、倒空那个桶。

插入任务时，只有这个桶的到期时间**变了**才重新入堆。同一个 tick 里后来的任务搭便车，堆的大小和「当前有任务的槽」同阶，不是和定时器个数同阶。

空闲时堆是空的，轮子完全不醒。这是时间轮能扛住海量空闲超时的另一半原因：成本发生在「有任务的格子」上，而不是发生在「墙上的每一毫秒」上。

```text
delay queue（按桶的到期时间）

  12:00:00.040  bucket[4]  → 到点 Flush，任务要么执行，要么降级
  12:00:00.080  bucket[1]  （上层）
```

堆只给桶排序，不给单个定时器排序。单个定时器仍然只是链表节点，增删 O(1)。

## 取消和重置

`Stop` 必须是 O(1)，否则取消比堆还贵，时间轮就没意义了。每个 `Timer` 记住自己所在的桶和链表节点，取消就是从链表摘掉。

并发下有个麻烦：时间轮的 goroutine 可能正在把这个 timer 从本层挪到下层（Flush → 再插入）。`Stop` 看见的桶可能已经不是现在的桶。做法是循环：摘一次，再读一次当前桶，直到它变成 nil——要么已经被摘走准备执行，要么已经被你摘干净。

`Reset(d)` 是「先 Stop，再按新的到期时间插回去」。和标准库一样，`Reset` 和「刚好到期」之间存在窗口：旧的回调可能已经进了 goroutine，新的调度也会生效。调用方不能假设 `Reset` 之后旧任务一定没跑。

`Ticker` 就是「到期后按固定间隔再插一次」。下一轮的时间从**上次的计划时间**算，不从任务跑完的时间算，慢任务不会把整条时间轴往后推，只会造成重叠。

## 一组对齐 time 包的接口

仓库 [seamonw/timingwheel](https://github.com/seamonw/timingwheel) 按上面的模型实现，接口尽量跟 `time` 对齐，方便从标准库换过来：

| 方法 | 对应 | 说明 |
| --- | --- | --- |
| `New(tick, wheelSize)` | — | 创建时间轮 |
| `Start` / `Stop` | — | 启动 / 停掉后台 goroutine |
| `After(d)` | `time.After` | 到期后向 channel 发送当前时间 |
| `AfterFunc(d, f)` | `time.AfterFunc` | 到期后在独立 goroutine 里调 `f` |
| `NewTimer(d)` | `time.NewTimer` | 带 `C`、`Stop`、`Reset` |
| `NewTicker(d)` | `time.NewTicker` | 带 `C`、`Stop`、`Reset` |
| `Tick(d)` | `time.Tick` | 只返回 channel，停不掉 |
| `Sleep(d)` | `time.Sleep` | 用时间轮阻塞当前 goroutine |
| `ScheduleFunc` | — | 按 `Scheduler.Next` 重复调度 |

最小例子：

```go
tw := timingwheel.New(time.Millisecond, 20)
tw.Start()
defer tw.Stop()

tw.AfterFunc(500*time.Millisecond, func() {
    fmt.Println("fired")
})

timer := tw.NewTimer(200 * time.Millisecond)
<-timer.C

ticker := tw.NewTicker(100 * time.Millisecond)
defer ticker.Stop()
<-ticker.C
```

`New(1*time.Millisecond, 20)` 第一层覆盖 20ms，第二层 400ms，第三层 8s。任务超出本层区间时现建上层，用不到的层不占槽位。

有几条和标准库不同、用之前要心里有数：

- **精度是 `tick`。** 到期时间对齐到 tick 边界，不适合亚毫秒级的准时调度。
- **回调不在轮子的 goroutine 里跑。** 一个慢任务不能堵住后面所有定时器。
- **`Stop` 之后调度的任务会被丢掉。** 轮子停了，桶不会再被倒空。
- **`Tick` 停不掉。** 和 `time.Tick` 一样，能用 `NewTicker` 就用 `NewTicker`。

## 实现上几个值得看的点

插入是整套算法的中心。本层放得下就落到 `(expiration/tick) % wheelSize`；否则 CAS 出一层溢出轮，tick 等于本层的 `interval`：

```go
func (tw *TimingWheel) add(t *Timer) bool {
    currentTime := tw.currentTime.Load()

    switch {
    case t.expiration < currentTime+tw.tick:
        return false // 已经到期
    case t.expiration < currentTime+tw.interval:
        virtualID := t.expiration / tw.tick
        b := tw.buckets[virtualID%tw.wheelSize]
        b.Add(t)
        if b.SetExpiration(virtualID * tw.tick) {
            tw.queue.Offer(b) // 到期时间变了才入堆
        }
        return true
    default:
        ow := tw.overflowWheel.Load()
        if ow == nil {
            tw.overflowWheel.CompareAndSwap(nil,
                newTimingWheel(tw.interval, tw.wheelSize, currentTime, tw.queue))
            ow = tw.overflowWheel.Load()
        }
        return ow.add(t)
    }
}
```

`SetExpiration` 用一次原子交换判断「这个桶是不是换了一轮」。同一轮里后面的任务不再碰延迟队列。桶到期后 `Flush`：先把链表摘空、把到期时间打回 -1，再逐个 `addOrRun`。摘空和再插入拆开，是为了让再插入可以落回这个桶本身，而不在同一把锁上死锁。

`Reset` 和 `Flush` 会抢同一个 `Timer`。实现里用一把 timer 自己的锁串起来：`Flush` 先在桶锁下摘节点，释放桶锁再 `addOrRun`；`Reset` 持 timer 锁摘节点再插回去。两边都看见「已经在某个桶里」就不再插第二份，避免同一个 timer 挂进两条链表。

延迟队列的等待不用 `time.After`。`time.After` 在超时前不会被 GC，循环里每次新建等于泄漏一个 runtime timer。这里是 `time.NewTimer`，被更早的插入唤醒时把 timer `Stop` 掉。

## 什么时候不该用时间轮

时间轮不是更准的 `time.Timer`，是更便宜的批量超时器。

- 定时器数量不大，标准库就好。实现更简单，精度只受系统时钟限制。
- 需要「刚好在这一纳秒」开火，时间轮做不到。它的世界是离散的。
- 延迟跨度极大、又要求每一档都很细，层次会变深，降级次数变多。这种时候往往该重新选 `tick` 和 `wheelSize`，而不是无脑加层。

Go 运行时最终没有用时间轮，选了按 P 分片的堆。原因也写在这段取舍里：堆能覆盖任意长度的延迟，不用处理降级，而且可以挂在调度器已有的唤醒点上，不必单独养一根指针。时间轮赢在「同质、海量、允许对齐误差」的那一类负载。连接超时就是这类负载的典型。

## 参考

- George Varghese, Tony Lauck. *Hashed and Hierarchical Timing Wheels: Efficient Data Structures for Implementing a Timer Facility.* SOSP 1987 / IEEE/ACM ToN 1997
- [Apache Kafka: Hierarchical Timing Wheels](https://www.confluent.io/blog/apache-kafka-purgatory-hierarchical-timing-wheels/)
- 实现仓库：[github.com/seamonw/timingwheel](https://github.com/seamonw/timingwheel)
