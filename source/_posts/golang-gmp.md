---
title: Go 调度器：G、M、P 与 goroutine 调度
date: 2026-09-02 16:30:00
categories: Golang
tags:
	- golang
	- scheduler
	- gmp
	- 并发
---

「goroutine 很轻，所以可以万并发。」创建便宜这半句是真的：Linux/amd64 上起始栈最小 2KB（`fixedStack`），`go f()` 不走一次 `clone`。「一万个同时往前跑」这半句是错的——同时执行 Go 代码的至多 `GOMAXPROCS` 个，其余在排队，或者在 channel、网络、系统调用上睡着。

调度器要回答的不是「怎么让一万个 goroutine 同时跑」，而是：**远多于 CPU 的 goroutine，怎么在少量线程上低开销地轮转，并且在其中一些被内核堵住时，不把 CPU 一起堵死。**

本文的事实以 Go 1.27 的 `runtime/proc.go` 为准，涉及版本差异的地方会标出。

## 「很轻」的账单

轻在栈小、创建和切换不进内核；账单出在元数据、栈增长、占线程的阻塞和纯 CPU 工作上。

| | OS 线程 | goroutine |
| --- | --- | --- |
| 创建 | `clone` / `CreateThread`，微秒级起 | `go f()`，一次内存分配量级 |
| 栈 | 通常预留 8MB 虚拟空间 | 起始 2KB 起，按需倍增拷贝 |
| 切换 | 进内核：保存寄存器、换调度实体 | 用户态：少量寄存器 + 回到 `schedule` |
| 同时执行 | 每条线程都是一个内核实体 | 上限是 P 的数量 |

不轻的四笔：

- **`allgs` 只增不减。** 创建过的 `g` 对象不释放，只进自由列表复用。峰值百万再降下来，元数据还挂着。
- **栈会涨，也会被丢。** 起始大小不是固定的 2KB：`startingStackSize` 每轮 GC 按近期平均栈大小上调。G 退出时若栈已不等于 `startingStackSize`，直接释放，下次重新分配。
- **阻塞可能占着 M。** 网络走 netpoller 不占线程；一个卡住的文件 `Read`、一个慢 cgo 调用会把 M 钉死，P 被别人拿走，进程线程数上涨。
- **真·CPU 工作不打折。** 一万个 G 各自算圆周率，仍然只有 `GOMAXPROCS` 个在跑，调度本身还要占周期。

所以「万并发」成立的前提几乎总是：**大多数 goroutine 大部分时间在等，而不是在算。**

## 为什么需要第三个实体

把 G 直接映射到线程的两种朴素做法都在规模上翻车，P 是为了修掉它们共同的那个缺陷。

| 模型 | 失败点 | GMP 的答案 |
| --- | --- | --- |
| 一对一：每个 G 一条 OS 线程 | 连接数 = 线程数，创建、切换、栈全按线程付；C10K 就死在这 | G 是用户态对象，数量与 M 解耦 |
| M:N + 单条全局队列（Go 1.0 那一代） | 每次 `go`、每次唤醒、每次再调度都抢 `sched.lock`；核越多锁越热，缓存行来回失效；刚唤醒的消费者还可能跑到别的核上 | 每张 P 一条本地队列 + `runnext` 插队槽 |
| 两者共有 | 线程被 syscall 挂起时，排在它身后的 G 一起停 | 调度权从线程上剥离，可 `handoffp` 转给别的 M |

第三行才是 P 的由来：**把「线程」和「跑 Go 代码的资格」拆成两样东西。**

## G、M、P：谁管什么

Go 1.1 起的调度器（设计见 [Scalable Go Scheduler Design](https://golang.org/s/go11sched)）用三个实体分摊职责。

```text
   M（OS 线程，数量可比 P 多）
   │  自带 g0 系统栈：schedule、扩栈、GC 辅助都在上面跑
   │  持有 P 才能执行用户 Go 代码
   ▼
 ┌────────────────────────────────────────────────┐
 │ P（门票 + 本地状态）                            │
 │   runnext · runq[256] · mcache · timer 堆      │
 └────────────────────────────────────────────────┘
   │ execute
   ▼
   G（goroutine）：栈、PC、状态（_Grunnable / _Grunning / _Gwaiting / _Gsyscall）
```

P 不是 CPU，是**跑 Go 代码所需的那张门票**：没有 P 的 M 只能卡在内核里，或者睡在空闲 M 列表上。

| | 数量 | 挂着什么 | 上限 |
| --- | --- | --- | --- |
| G | 十万级无妨 | 栈、PC、等待项 | 内存 |
| M | 随「同时卡住的内核调用数」增长 | g0、信号栈、当前 G | `sched.maxmcount`，`schedinit` 里写死 10000，越界 `throw("thread exhaustion")` |
| P | `GOMAXPROCS` | 本地队列、`mcache`、timer 四叉堆 | 等于真并行度 |

只有 G 和 M 的话，本地队列和分配器缓存就得挂在 M 上：线程一进 syscall 被内核挂起，这些状态跟着沉睡，别人用不了。P 让线程可以睡，门票转给醒着的人。

`GOMAXPROCS` 从 Go 1.5 起默认等于逻辑 CPU 数。**Go 1.25 起，Linux 上默认还会考虑 cgroup CPU 限额**，并由 `sysmon` 每秒最多更新一次（`GODEBUG=containermaxprocs=0` 关掉 cgroup 感知，`updatemaxprocs=0` 关掉动态更新）；1.24 及更早只看 CPU 数和亲和性，64 核机器上给 2 核配额的容器照样拿到 64 张 P。改它是在改**并行度**，不是在改「能起多少 goroutine」。

## 三处可运行队列

每张 P 有三处放可运行 G 的地方，快路径完全不碰全局锁。

```text
 go f() / goready(gp)
        │
        │ next=true                    runqput 慢路径（本地满）
        ▼                                     │
  ┌─ runnext ─┐  1 个槽，runqget 先看它        │  runqputslow：队头 128 个
P─┤           │                               ▼  + 当前这个 = 129 个一次搬走
  └─ runq[256] 环形，FIFO，本 P 无锁存取 ──▶ 全局 runq（sched.lock 保护）
```

容量 256 是写死的；溢出不逐个进全局队列，而是一次搬 129 个——摊薄锁开销，同时留一半在本地保住局部性。

`runnext` 只有 1 个槽，因为它解决的是一个很窄的问题：

- 生产者-消费者太常见。`ch <- v` 唤醒的对端立刻在同一张 P 上跑，数据还在 L1；丢到队尾的话，中间可能插进几十个不相干的 G。
- `execute` 从 `runnext` 取 G 时继承当前时间片（`inheritTime`），不推进 `schedtick`——当成同一段工作的延续，而不是一次新调度。
- 它不是优先级队列：只看「是不是刚被这张 P 造出来或喊醒」，不看 goroutine 有多重要。

## findRunnable 的查找顺序

公平性和局部性的取舍全写在这个顺序里，`findRunnable` 每次调度都按它走一遍：

```text
findRunnable()
 1. trace reader / GC worker                      有标记任务先领
 2. pp.schedtick%61 == 0 && 全局队列非空 → 取 1 个   ← 写死的公平性阀门
 3. 本地：runqget(pp)                             runnext → runq 队头
 4. 全局：globrunqgetbatch(len(runq)/2)            一次搬 128 个进本地
 5. netpoll(0) 非阻塞                             有 waiter 且没人正在 poll 时才做
 6. stealWork()                                   4 轮随机扫 allp
 7. GC idle mark                                  能干标记活就不还 P
 8. 都没有 → 交还 P、stopm                        必要时做一次阻塞 netpoll
```

第 2 步的 61 防的是饿死：两个 G 不停 `go` 对方就能永远占满本地队列，偶尔强制从全局取一个，溢出去的工作才有机会跑。第 8 步那次阻塞 netpoll 用一次睡眠同时等网络和最近的 timer。

## 工作窃取：偷一半，随机起点

本地队列消掉了锁竞争，代价是不平衡；窃取用「一次搬一半」把再平衡的成本摊薄。

```text
偷之前                            偷之后（runqgrab: n = n - n/2，从队头那端拿）
P1 runq: [g1 g2 g3 g4 g5 g6]      P1 runq: [g4 g5 g6]      ← 较新的留给原主
P2 runq: []（空转中）             P2 runq: [g1 g2] ＋ 直接开跑 g3
```

偷 1 个则下次还得再来，跨核 cache miss 摊不薄；偷光则对方立刻回头偷你，来回抖动。取一半让双方都有活干。

- **从队头那端拿。** 那些入队最早，和原主当下正在做的事关系最远。
- **`runnext` 最后才考虑。** 队列里没东西可偷时才动它，且先 `usleep(3)`（低精度定时器平台上改成 `osyield`）让原主有机会自己调度它，避免同一个 G 在两张 P 之间被推来推去。
- **随机起点，不是 0、1、2 顺序扫。** `stealOrder` 用与 P 个数互质的步长走一个伪随机排列，顺序扫会让所有闲 P 挤向同一张忙 P。
- **spinning M 有配额。** 条件是 `2*nmspinning < gomaxprocs - npidle`，即 spinning 的 M 不超过忙碌 P 的一半，免得 `GOMAXPROCS` 很大而实际并行度很低时一堆线程烧 CPU。

配套的是 `wakep`：只在「有空闲 P 且当前没有 spinning M」时才喊醒线程；而那条找到活、退出 spinning 的 M 有责任再补一个 spinning，否则会出现「大家都以为有人在找、其实没人在找」的空窗。spinning 是 M 在 `findRunnable` 里多看几眼别人的队列和 timer，不是用户 goroutine 在空转。

## syscall：P 会被夺走

用户代码里的「阻塞」不是一种。调度器只对**会卡住 OS 线程的那种**做特殊处理。

```text
时间 ↓   M1                              P                        另一条 M
       entersyscall             _Prunning → _Psyscall
       （记下 oldp）            从 M1 摘下，本地队列原样留着
       陷入内核 read ……
                                ← sysmon 的 retake 巡到
                                   handoffp(P) ──────────▶ M2 绑上 P，接着跑它的本地队列
       read 返回
       exitsyscall：试 oldp，失败
       → 申请空闲 P；拿不到就把 G 丢进全局队列，自己 stopm
```

卡住的是线程，不是这颗核上的 Go 调度：P 的数量不变，M 的数量跟着**同时卡住的内核调用数**涨。两条状态线并排看更清楚：

```text
P：_Prunning ──entersyscall──▶ _Psyscall ──retake(CAS)──▶ _Pidle ──handoffp──▶ _Prunning（在 M2 上）
G：_Grunning ────────────────▶ _Gsyscall ──────────────────────────────────▶ _Grunnable（回队列排）
M1：跑 Go 代码 ──────────────▶ 卡在内核，无 P ─────────────────────────────▶ 抢 P 或 stopm
```

G 全程没有丢，只是换了一条线程继续；M1 在这段时间里对调度器而言等于不存在。

- **短 syscall 赌快路径。** `exitsyscall` 优先用 `oldp` 把同一张 P 抢回来，`mcache` 和局部性都还在。
- **`retake` 的条件。** P 在 syscall 里跨过一个 sysmon tick（至少 20µs）后，只要「本地队列非空」、或「没有空闲/spinning 的 M 能顶上」、或「已经堵了 10ms」，就夺走并 `handoffp`。
- **明知会堵的走 `entersyscallblock`**，立刻 `handoffp`，不等 sysmon 来收。
- **`handoffp` 负责兜底**：如果这张 P 还有活、全局还有活、或需要有人去 netpoll，就必须有 M 接手，没有闲 M 就 `newm` 造一条。
- **cgo 走同一套门。** `entersyscall` → C 代码 → `exitsyscall`。C 里没有抢占点，这段时间运行时几乎看不见。

## sysmon 与抢占：从协作到信号

时间片 10ms（`forcePreemptNS`）是 `sysmon` 的愿望；G 真正让出 P，要么自己 `gopark`，要么在一个安全点被按回调度器。

`sysmon` 由 `newm(sysmon, nil, -1)` 在启动时拉起，不持有 P，不参与 `schedule` 循环：睡 20µs 起，连续空转约 1ms 后指数加倍，上限 10ms。每轮扫一遍 `allp`，做 `retake`、抢占跑满时间片的 G、超过 10ms 没 poll 就补一次 `netpoll`、查死锁、催 GC，1.25 起还顺手更新 `GOMAXPROCS`。

抢占请求怎么送到，1.14 是分界线：

| | 请求方式 | 生效位置 | 热循环 |
| --- | --- | --- | --- |
| 1.14 前（协作式） | `gp.stackguard0 = stackPreempt` | 函数序言的栈溢出检查误判，走进运行时 | 抢不动 |
| 1.14 起（＋异步） | 追加一个 `SIGURG`（`sigPreempt`） | 信号处理器在 `gsignal` 栈上改返回地址，注入 `asyncPreempt` | 能打断 |

「抢不动」的后果很具体：

```go
func freeze() {
    n := 0
    for {
        n++ // 没有函数调用，就没有序言，看不见 stackguard0 哨兵
    }
}
```

1.14 之前这个 G 会一直占着那张 P，而 GC 的 STW 要等所有 P 到达安全点——**整个程序冻在一个没有函数调用的热循环上**，这是当年生产环境里真实出现过的事故。1.14 之后信号能改 PC，多数用户代码可被异步打断；但运行时内部、部分汇编、持有运行时锁的区间仍标记为不可抢占。`GODEBUG=asyncpreemptoff=1` 只用于排查，不是调优手段。

## netpoller：等网络不占 M+P

网络是 goroutine 最多的那种「等」，它必须走 park 而不是走 syscall 路径，否则 GMP 退回一对一。

```text
conn.Read(buf)
 ├─ 非阻塞 recv 有数据 → 直接返回，一次调度都不发生
 └─ EAGAIN：
      netpoller 登记 fd → G   （Linux epoll / BSD kqueue / Windows IOCP）
      gopark：G 变 _Gwaiting，从队列里消失
      M 回到 findRunnable，P 没动 ──▶ 这张门票继续跑别人
    fd 就绪：netpoll 取回 G → goready → 进某张 P 的队列
```

等 I/O 的 G 占着的只有 epoll 里一项和一份小栈，这是「每连接一个 goroutine」能上万的全部秘密；非阻塞是前提，细节见 [为什么 IO 多路复用必须搭配非阻塞 IO](/blog/2026/09/16/io-multiplexing-nonblocking/)。

四种等待的账单完全不同：

| 等待 | G | M | P |
| --- | --- | --- | --- |
| channel、竞争中的 `sync.Mutex` | `gopark` → `_Gwaiting` | 换下一个 G | 不动 |
| `net.Conn` 读写、Listener | `gopark`，挂进 netpoller | 换下一个 G | 不动 |
| `time.Sleep`、deadline | `gopark`，挂进 P 的 timer 堆 | 换下一个 G | 不动（timer 可被窃取最后一轮取走） |
| 阻塞文件 syscall、慢 cgo | `_Gsyscall` | 卡在内核 / C 里，线程数 +1 | `_Psyscall`，可被 `retake` |

前三行是主路径，第四行是例外。`net/http` 服务几乎只走前三行；自己拿 `os.File` 对套接字做阻塞读、或在 cgo 里 `poll`，才会掉进第四行。

## 一次请求在调度器里怎么走

`GOMAXPROCS=4`，一个「读 socket → 读本地文件 → 回包」的 handler，在调度器里的轨迹是：

```text
accept 的 G park 在 netpoller
 └─ 连接到来 → goready → 某张 P → execute
     └─ go handle(conn)：新 G 进当前 P 的 runnext，下一拍就跑，不绕全局队列
         ├─ conn.Read 无数据 → gopark；M+P 立刻跑本地队列里下一个
         ├─ 数据到达 → findRunnable 第 5 步捞回来（理想情况回到原来那张 P）
         ├─ os.File.Read 读盘 → entersyscall；超过 sysmon 的耐心 → retake + handoffp，线程数 +1
         ├─ 回包：非阻塞 write，多半不堵线程
         └─ G 退出 → _Gdead；栈若不等于 startingStackSize 就释放，g 进 gFree，allgs 不缩
```

同一时间旁边那个算 JSON 的 G 连续占满 10ms，会被 `stackPreempt` + `SIGURG` 按回队列，P 转去跑刚从 netpoller 醒来的 handler。整条链路上真正占着 P 的，只有「正在跑 Go 代码」的那些瞬间。

## 避坑

**1. 忙等。** `for { if atomic.Load(&flag) == 1 { break } }` 在 1.14 之前能卡死 STW，之后也要吃满一个 10ms 时间片。有等待语义就用 channel 或 `Mutex`；`runtime.Gosched()` 能「治好」的问题，说明原语选错了。不可抢占的热循环还会拖长 STW 的停机阶段：`/sched/pauses/stopping/gc:seconds` 的分位数逼近 `/sched/pauses/total/gc:seconds` 时，时间就是花在等 P 到达安全点上。

**2. 线程数无端上涨。** 连接数和 `GOMAXPROCS` 都没变，`top` 里线程数却在涨，答案基本在阻塞文件 I/O 和 cgo：P 被 `handoffp` 走了，总得有条线程来绑它。涨到 `maxmcount` 会直接 `throw("thread exhaustion")`。排查顺序是 `GODEBUG=schedtrace=1000`（看 `threads` 与 `idlethreads`）→ pprof 的 threadcreate → 数 cgo 调用点。

**3. 把 `GOMAXPROCS` 当连接配额。** 它是并行度上限，不是 goroutine 配额；调小它是在限 CPU，不是在限连接。判断「是 G 太多还是 P 太少」看排队时间，别看 `NumGoroutine()`：

```go
import "runtime/metrics"

var samples = []metrics.Sample{
	{Name: "/sched/goroutines:goroutines"},
	{Name: "/sched/gomaxprocs:threads"},
	{Name: "/sched/latencies:seconds"}, // 直方图：G 进入 _Grunnable 到真正开跑的等待，Go 1.20 起
}

// p99 明显高于个位数微秒，说明可运行的 G 在排队等 P，不是在等 I/O。
func schedPressure() (live, procs uint64, p99 float64) {
	metrics.Read(samples)
	// histQuantile 略：按 Float64Histogram 的 Counts / Buckets 累加到目标分位。
	return samples[0].Value.Uint64(), samples[1].Value.Uint64(),
		histQuantile(samples[2].Value.Float64Histogram(), 0.99)
}
```

较新的运行时还提供 `/sched/threads/total:threads` 和 `/sched/goroutines/not-in-go:goroutines`，正好对应第 2 条；用 `metrics.All()` 先确认当前版本有没有这两项。

## 结论

GMP 是一次职责拆分：G 是可以很多、可以睡的工作，M 是能被内核执行的线程，P 是跑 Go 代码的门票和本地缓存。队列分片为了不上全局锁，`runnext` 为了把「刚喊醒」当成一次函数调用，偷一半为了核别闲着，`handoffp` 为了线程卡住时门票继续转，`sysmon` 加 `SIGURG` 为了热循环和慢 syscall 停不住世界，netpoller 为了让「等网络」回到 park。

- G 远多于 P 成立的前提是它们在**等**；一万个 `for { n++ }` 会把 P 吃满，尾延迟和 GC STW 一起上来。
- 抢占保的是系统整体推进，不是你的尾延迟——调度器没有优先级，重活放错地方就只能自己承担。
- 业务侧什么时候该停手不再 `go f()`，见 [Go 任务池：并发限制、排队与退出](/blog/2026/09/14/golang-worker-pool/)；取消和超时怎么往下传，见 [Go context：设计、源码与代价](/blog/2026/09/02/golang-context/)。
