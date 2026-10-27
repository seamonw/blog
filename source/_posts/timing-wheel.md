---
title: 时间轮算法：原理、层次化实现和一组 Go Timer 接口
date: 2026-08-28 19:30:00
categories: 算法
tags:
	- 时间轮
	- golang
	- timer
	- kafka
	- netty
	- 面试
---

定时器看起来是件小事：睡一会儿，到点做事。数量少的时候，标准库 `time.Timer` 完全够用。数量到十万、百万——连接空闲超时、心跳、重试退避、延迟消息——问题就变成另一件事：怎么让「加一个、取消一个」的代价不跟着定时器总数一起涨。

时间轮就是为这个场景准备的。本文先把单层轮、圈数和层次化轮讲清楚，再对照一份 Go 实现说明 `Timer` / `Ticker` 怎么挂上去。文末是面试里要能当场算出来的重点，以及三处最容易差一圈的难点。完整代码在 [seamonw/timingwheel](https://github.com/seamonw/timingwheel)。

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

**覆盖范围有限。** 不记圈数的话，8 个 1 秒槽只能表示 8 秒以内的延迟。要表示一天，要么把 `tick` 拉大（精度烂掉），要么把槽位加到 86400（内存和空转都难看）。

**空转。** 指针每个 tick 都要醒一次。轮子上只有几个任务时，你在为空气付 goroutine 唤醒。

范围可以用「圈数」补，不必先上层次。Netty 的 `HashedWheelTimer` 就是这样：任务除了槽位，还记 `remainingRounds`。指针每次转到这个槽，圈数大于 0 就减一，减到 0 之后的下一次再转到此槽才执行。

```text
wheelSize = 8，槽 4 上的链表。指针每转到槽 4 都要整表扫一遍

  [A rounds=0 本圈执行] -> [B rounds=12 每圈只减一] -> [C rounds=0]
```

圈数把一天的超时塞进有限个槽里了，远期任务却还挂在细轮上。槽里若有十万个「明天」的连接超时，每一圈都把这十万个节点碰一遍，只为了给圈数减一。插入仍是 O(1)，触发变成 O(槽内链表长度)。

层次化解决的是这次扫描：远期任务放进粗轮，细轮只留快到期的。延迟队列解决空转。

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

Kafka 的请求超时走层次化轮。Netty 的 `HashedWheelTimer` 停在单层加圈数：默认 100ms 一拍、512 槽，一圈 51.2 秒，更长的延迟靠 `remainingRounds`，代价就是上一节的整表扫描。层次化把精度、覆盖范围和扫描成本拆开。

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

Go 运行时没有用时间轮。截至 Go 1.24，`src/runtime/time.go` 里每个 P 仍是一棵四叉堆（`timerHeapN = 4`）。堆能覆盖任意长度的延迟，不用处理降级，而且可以挂在调度器已有的唤醒点上，不必单独养一根指针。时间轮赢在「同质、海量、允许对齐误差」的那一类负载。连接超时就是这类负载的典型。

## 面试重点

背「插入 O(1)」不够。下一句一定是：代价是什么、圈数怎么算、和堆比你选哪个。

| 追问 | 答到这里可以停 |
| --- | --- |
| 和最小堆比，赢在哪 | 插入和取消 O(1)，堆是 O(log n)。n = 10⁶ 时 log₂n ≈ 20，堆并不慢。时间轮赢在取消远多于触发，以及不用维护全局有序 |
| 槽位怎么算 | `slot = (expiration / tick) % wheelSize`。先判断到期时间落不落在本层 `interval = tick × wheelSize` 里，落不下就进上一层 |
| 单层怎么覆盖长延迟 | `remainingRounds`。范围补上了，扫描没补上：指针每过这个槽都要遍历整条链表 |
| 层次化怎么降级 | 上层桶到期后不执行，按绝对到期时间从最细的轮重新 `add`。真正开火只发生在最细一层 |
| 空转怎么办 | 延迟队列里排的是桶的到期时间，不是单个任务。空闲时堆是空的，轮子不醒。堆的大小正比于有任务的槽 |
| 取消为什么必须 O(1) | timer 记住自己的桶和链表节点，`Stop` 是摘节点。做不到这一点，时间轮就退回堆 |
| 任务有序吗 | 没有全局到期序。槽与槽之间靠指针，槽内靠链表。同一 tick 的任务一起到期 |
| 精度是多少 | 误差上界是一个 tick。方向取决于对齐到槽的起点还是终点 |
| 什么时候改用堆 | 数量不大、延迟跨度很散、要贴近系统时钟。Go 1.24 运行时就是每个 P 一棵四叉堆 |

## 面试难点

三处最容易在白板上差一圈，或者把误差方向讲反。

### 圈数不是 delay / wheelSize

Netty 4.1 在 worker 把任务从 MPSC 队列搬进桶时计算。`tick` 是轮子已经走过的拍数，`deadline` 是相对 `startTime` 的纳秒：

```java
calculated = deadline / tickDuration;
remainingRounds = (calculated - tick) / wheel.length;
stopIndex = max(calculated, tick) & mask; // 槽数做成 2 的幂，取模变成按位与
```

转到该槽时，`remainingRounds > 0` 就减一，减到 0 的下一次再转到此槽才执行。减圈和执行不在同一次访问里。

`wheelSize = 8`，当前 `tick = 3`，`calculated = 8`：

```text
正确：rounds = (8 - 3) / 8 = 0，槽 = 8 & 7 = 0
      下次处理槽 0 是 tick = 8，圈数已是 0，执行

错误：rounds = 8 / 8 = 1，槽 = 0
      tick = 8 只把圈数减到 0，tick = 16 才执行，晚一整圈
```

`max(calculated, tick)` 是另一半。漏掉它，已经错过的任务会被放进指针走过的槽，再等一整圈。默认一圈 51.2 秒，公式差 1，超时就差一圈。

### 误差方向：Kafka 偏早，Netty 偏晚

两边的误差界都是一个 tick，方向相反。

Kafka 把桶的到期时间设成 `floor(expiration / tick) × tick`，也就是槽的起点。延迟队列在这个时刻倒桶，任务从最细的轮重新 `add`；`expiration < now + tick` 就立刻执行。到期时间落在槽的后半段时，会提前将近一个 tick。

`tick = 10ms`，本层 `currentTime = 200ms`，`interval = 100ms`，任务到期 `259ms`：

```text
259 < 200 + 100，留在本层
virtualId = 259 / 10 = 25，槽 = 25 % 10 = 5
桶到期 = 25 × 10 = 250ms

t = 250 时重新 add：259 < 250 + 10，判定已到期，执行
比 259 早 9ms
```

Netty 在每一拍的结尾醒来，`waitForNextTick` 睡到 `tickDuration × (tick + 1)`。任务最早也要等这一拍走完，所以是晚不到一个 tick。worker 若被回调堵住，晚的时间没有上界。回调不能放在轮子线程里跑，原因就在这里。

不要说「时间轮是精确的」，也不要说「都会提前」。先问对齐到槽的哪一头。

### 轮子不是并发数据结构

两条路，混用就会丢任务，或者同一个 timer 触发两次。

**外部只投队列，轮子归一根线程。** Netty 的 `newTimeout` 只把任务放进 MPSC 队列，取消放进另一条队列。改链表、减圈数、触发都在 worker 上，用户线程不拿桶锁。代价是入桶有延迟：每个 tick 最多搬 100000 个新任务，突发创建会拖到后面的拍。

**外部直接摘链表。** 本文的 Go 实现走这条。`Flush` 正在把 timer 降级到下层时，另一个 goroutine 可能正在 `Stop` 或 `Reset`。桶锁里只做摘链，放下桶锁再 `add`。在桶锁里降级插入，两把桶锁会交叉死锁。`Reset` 用 timer 自己的锁串起「摘下来、再插回去」，避免同一个 timer 挂进两条链表。`Stop` 要循环摘，直到看到的桶变成 nil：你摘的那一下，它可能刚被挪到下一层。

讲并发时先说清楚谁拥有槽上的链表。

## 总结

时间轮把「维护全局有序」换成「下一个 tick 扫哪个槽」。单层加圈数补范围，层次化把远期任务移出细轮，延迟队列让空槽不再醒来。插入和取消是 O(1)，精度是一个 tick，触发要付槽内链表的钱。连接超时这类又多又常取消的任务适合它；要贴近系统时钟，或者数量不大，用堆。

## 参考

- George Varghese, Tony Lauck. *Hashed and Hierarchical Timing Wheels: Efficient Data Structures for Implementing a Timer Facility.* SOSP 1987 / IEEE/ACM ToN 1997
- [Apache Kafka: Hierarchical Timing Wheels](https://www.confluent.io/blog/apache-kafka-purgatory-hierarchical-timing-wheels/)
- 实现仓库：[github.com/seamonw/timingwheel](https://github.com/seamonw/timingwheel)
