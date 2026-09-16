---
title: Go 任务池：并发限制、排队与退出
date: 2026-09-14 20:12:00
categories: Golang
tags:
	- golang
	- worker-pool
	- 并发
	- 调度器
---

「goroutine 很轻，所以不需要线程池」和「所以可以随便 `go`」是两句经常连在一起说的话。前半句对：Go 没有 OS 线程池那种「创建太贵必须复用」的压力。后半句是错的。任务池在 Go 里要管的不是创建开销，是**同时在飞的工作有多少、多出来的去排队还是被丢掉、失败了怎么停**。

调度器怎么把 G 轮转到 P 上，已经写过一篇：[Go 调度器：G、M、P 与 goroutine 调度](/blog/2026/09/02/golang-gmp/)。那篇回答的是运行时怎么跑；本文回答的是业务侧什么时候不该再 `go f()`。取消怎么往下传，见 [Go context：设计、源码与代价](/blog/2026/09/02/golang-context/)。

按四条线走：

1. 账单出在哪，而不是「起一个 G 要几纳秒」；
2. 三种常见限流形状：信号量、固定 worker、可回收 worker；
3. 一份带 `context`、panic 恢复和优雅退出的极简实现；
4. 调参、死锁和什么时候不该用池。

## 为什么「很轻」仍要池

一次 `go f()` 大约是一次堆分配量级。起始栈约 2KB，不进内核。这一点没有变。变贵的是规模和形态：

- **`g` 会留在 `allgs` 里。** 创建过的 goroutine 对象，退出后进空闲列表复用，但这份元数据不会随着负载降下来而还给 OS。突发一百万再降到一百，扫描 `allgs` 的代价还在。
- **栈会涨。** 任务里有深调用、大框架，栈从 2KB 拷到更大。涨过起始大小的栈，G 退出就丢掉，下次再分配。池化 worker 把「热点栈」留在少数长期活着的 G 上，对这种任务更友好。
- **调度器和 GC 看见的是数量。** 每个可运行的 G 都要进某个 P 的本地队列或全局队列；每个活着的 G 的栈都是 GC 根。十万个只跑 50µs 的任务，创建本身不疼，队列和扫描会疼。
- **下游比 CPU 脆。** 一万个 goroutine 同时打 Redis、打磁盘、打另一个 RPC，先爆的是连接数、fd、对面的 QPS 上限，不是你的 `GOMAXPROCS`。

所以 Go 里的任务池，本质几乎总是**并发度控制器**，偶尔才是 **G 复用器**。Java 线程池是因为线程贵；Go 任务池是因为「无界并发」会把进程和下游一起拖死。

一个会在生产里出现的形状：

```go
func handle(w http.ResponseWriter, r *http.Request) {
    for _, id := range r.URL.Query()["id"] {
        go fetch(id) // 每个 query 参数一个 G，每个 G 再打下游
    }
}
```

QPS 100、每个请求 50 个 id，就是每秒 5000 个新 G、5000 个出站。运行时扛得住创建，下游和内存不一定。池把「同时 fetch 的上限」变成一个显式数字。

## 三种限流，不是一种池

名字都叫 pool，结构差很远。选错形状，调参没有意义。

### 1. 信号量：每个任务仍是一个 G，只是同时跑的有上限

```go
sem := make(chan struct{}, 8) // 最多 8 个在跑

func submit(fn func()) {
    sem <- struct{}{}
    go func() {
        defer func() { <-sem }()
        fn()
    }()
}
```

更干净的版本是 `golang.org/x/sync/semaphore` 或 `errgroup.Group` 的 `SetLimit`：

```go
g, ctx := errgroup.WithContext(parent)
g.SetLimit(8)
for _, job := range jobs {
    job := job
    g.Go(func() error {
        return do(ctx, job)
    })
}
return g.Wait()
```

特点：

- 实现成本接近零，标准库就能写。
- **每个任务仍创建/复用一个 G。** 不解决 `allgs` 膨胀，只解决「同时执行数」。
- `submit` 在槽满时会阻塞调用方。这对「入口已经是请求 goroutine」通常是对的：反压回到上游。
- 适合：任务时长差异大、数量不是百万级、你只想要一个并发上限。

`SetLimit` 从 Go 1.20 起就在 `errgroup` 里。一批「跑完这组任务就结束」的工作，优先用它，不要先写 worker 循环。

### 2. 固定 worker：N 个长期 G + 一条任务队列

```text
调用方 --Submit--> jobs chan --N 个 worker--> 执行
```

并发度等于 worker 数。多出来的任务进 channel。channel 容量就是队列长度：

- 容量 0：提交和执行握手，调用方被执行速度直接堵住。
- 容量很大：调用方几乎不堵，内存和延迟在队列里涨。
- 满了怎么办：阻塞、返回 `ErrPoolFull`、或者丢掉最旧的。这是产品决策，不是实现细节。

适合：任务同质、长期存在的后台流水线（压缩、写盘、推送）、需要稳定的并发度和排队策略。

### 3. 可回收 worker：队列空了就把 G 放回，超时再杀掉

ants 这类库走的是这条路：任务来了，空闲 worker 直接接；没有空闲就扩到 `Max`；空闲超过 `Expiry` 再退出。底层常常再用 `sync.Pool` 缓存 worker 结构体。

它同时做两件事：限并发，以及**让 G 的数量跟着负载涨跌**，避免固定 1024 个 worker 在夜里空转、在高峰又不够。空转的代价不是 CPU——阻塞在 channel 上的 G 几乎不占 P——而是那 N 份栈和 `g`。对大多数服务，固定 worker 已经够；流量脉冲极大时，动态扩缩才划算。

对照一张表：

| | 信号量 / errgroup | 固定 worker | 动态 worker |
| --- | --- | --- | --- |
| 每个任务一个新 G | 是 | 否 | 忙时扩，闲时收 |
| 排队 | 卡在获取槽位 | 卡在 channel | 卡在 channel 或直接拒 |
| 生命周期 | 这批任务 | 进程级 | 进程级，可过期 |
| 适合 | 一次扇出 | 稳定流水线 | 突发 + 长期服务 |

后面的实现用固定 worker：结构最清楚，也最容易把退出和 panic 说完。动态扩缩是在它上面加「空闲超时 + 再拉起」。完整代码在 [seamonw/workerpool](https://github.com/seamonw/workerpool)，仓库里用一把锁堵住了文中写到的「关 `jobs` 与发送」竞态。

## 极简任务池

最小接口只要四个动作：提交、尝试提交、等在飞任务结束、关掉池。

```go
package pool

import (
	"context"
	"errors"
	"sync"
)

var (
	ErrPoolClosed = errors.New("pool: closed")
	ErrPoolFull   = errors.New("pool: full")
)

type Pool struct {
	jobs    chan func()
	wg      sync.WaitGroup
	once    sync.Once
	closed  chan struct{}
}

func New(workers, queue int) *Pool {
	if workers <= 0 {
		workers = 1
	}
	p := &Pool{
		jobs:   make(chan func(), queue),
		closed: make(chan struct{}),
	}
	p.wg.Add(workers)
	for i := 0; i < workers; i++ {
		go p.worker()
	}
	return p
}

func (p *Pool) worker() {
	defer p.wg.Done()
	for fn := range p.jobs {
		run(fn)
	}
}

func run(fn func()) {
	defer func() {
		if err := recover(); err != nil {
			// 生产里这里打栈、加 metrics。绝不能让 panic 干掉 worker。
			_ = err
		}
	}()
	fn()
}
```

`recover` 不是装饰。固定 worker 模型下，一个任务 panic 且不恢复，这条 worker 就没了，池的实际并发度悄悄减一，直到变成零。信号量模型里更糟：`sem` 若在 panic 之后才释放，槽位泄漏，池永久堵死。所以 `run` 必须 `defer recover`，信号量必须 `defer` 释放。

提交要把「池已经关了」和「队列满了」分开：

```go
func (p *Pool) Submit(ctx context.Context, fn func()) error {
	select {
	case <-p.closed:
		return ErrPoolClosed
	default:
	}
	select {
	case <-p.closed:
		return ErrPoolClosed
	case <-ctx.Done():
		return ctx.Err()
	case p.jobs <- fn:
		return nil
	}
}

func (p *Pool) TrySubmit(fn func()) error {
	select {
	case <-p.closed:
		return ErrPoolClosed
	case p.jobs <- fn:
		return nil
	default:
		return ErrPoolFull
	}
}

func (p *Pool) Close() {
	p.once.Do(func() {
		close(p.closed)
		close(p.jobs)
	})
}

func (p *Pool) Wait() { p.wg.Wait() }
```

几个实现上的坑：

**先关 `closed` 再关 `jobs`。** 只关 `jobs` 的话，并发 `Submit` 可能往已关闭的 channel 发送而 panic。先让 `Submit` 看见 `closed`，再关队列，worker 在 `range` 里把剩余任务抽干后退出。`Close` + `Wait` 就是优雅退出：不再接新的，把队列里的做完。

**`Submit` 的双重 `select`。** 若只有第二个 `select`，`jobs` 一直能接收时，`closed` 和 `ctx.Done()` 可能永远轮不到——Go 的 `select` 在多个 case 都就绪时均匀随机，通常没事，但队列空闲时关闭探测会变慢。先非阻塞探一次 `closed`，再进入会阻塞的 `select`，关闭路径更干净。仍有极窄的竞态：探完之后、送进 `jobs` 之前池被关掉。`Close` 先关 `closed` 再关 `jobs`，后一个 `select` 仍能从 `closed` 返回。若 `close(p.jobs)` 已经发生，`p.jobs <- fn` 会 panic，所以 **任何 `Submit` 都不该在 `jobs` 已关之后还走到发送**。上面的顺序保证：`closed` 一定先于 `jobs` 关闭，发送 case 和 `closed` case 同时就绪时可能仍随机到发送——这是这个极简版的裂缝。

补上这道缝的办法是禁止在关了之后发送：用一把 `mu` 把 `closed` 标志和发送做成临界区，或只允许一个 goroutine 负责把任务塞进队列。生产库（ants、tunny）都有这把锁或 `atomic` 状态机。教材版把裂缝写出来，是因为「两个 close 的顺序」经常被当成已经足够。

**任务闭包会抓住外层循环变量。** 和普通 `go func()` 一样：

```go
for _, job := range jobs {
    job := job
    _ = p.Submit(ctx, func() { handle(job) })
}
```

少写那一行 `job := job`，池里跑的全是最后一项。Go 1.22 起 `for` 的循环变量每轮是新的，但池的代码经常要在 1.21 的机器上跑，显式 shadow 仍然值得留。

### 带结果和取消

`func()` 不够用时，不要把池改成泛型 `Pool[T]` 先。先让任务自己闭包住 `chan` 或 `errgroup`：

```go
func SubmitErr(p *Pool, ctx context.Context, fn func() error) <-chan error {
    ch := make(chan error, 1)
    err := p.Submit(ctx, func() {
        ch <- fn()
    })
    if err != nil {
        ch <- err
    }
    return ch
}
```

真正要取消**正在跑**的任务，只能靠任务自己听 `ctx`。池能取消的是「还没进 worker 的排队」：`Submit` 在 `ctx.Done()` 时放弃入队。已经在跑的 `fn`，框架插不进去——这和「不能从外面 Kill goroutine」是同一条约束，[Go context：设计、源码与代价](/blog/2026/09/02/golang-context/)写过原因。所以 `fn` 的签名更好是 `func(ctx context.Context)`，创建池的人把请求级 ctx 传进去，任务在 I/O 点检查。

```go
func (p *Pool) SubmitCtx(ctx context.Context, fn func(context.Context)) error {
    return p.Submit(ctx, func() { fn(ctx) })
}
```

注意：入队时的 `ctx` 和执行时的 `ctx` 是同一个。调用方超时后，排队中的任务应被丢掉（`Submit` 返回 `ctx.Err()`），执行中的任务应在下一处 `select` 看到 `Done` 后收尾。若 `fn` 是纯计算、中间不看 ctx，超时对它无效——池解决不了这件事。

## 怎么定 workers 和 queue

没有万能公式，但有方向。

**CPU 密集**（哈希、压缩、编解码）：worker 数在 `GOMAXPROCS` 附近。再多只会增加切换。队列可以短，反压回调用方比堆积更好；堆积只会让延迟从「排队」变成「超时」。

**I/O 密集**（RPC、磁盘、数据库）：worker 可以明显高于 `GOMAXPROCS`，因为大多数时间 G 睡在 netpoller 上，不占 P。上限看下游：连接池大小、对端 QPS、本机 fd。经验上先用「下游连接上限」当 worker 上限，而不是拍一个 1024。

**队列长度**决定的是延迟预算，不是吞吐。吞吐的上限是 worker 处理速度。队列 = 1000、单任务 10ms、8 个 worker，最坏排队约 `1000/8*10ms ≈ 1.25s`。这个数如果已经大于接口 SLA，队列就该更短，满了用 `TrySubmit` 返回 503，而不是让用户等 1.25s 再失败。

可以按利特尔法则心算：`L = λW`。目标延迟 W、到达率 λ，在途任务 L 不应超过 `λW`。worker 数是同时执行的部分，其余是排队。

**不要按「机器还能起多少 G」定。** 那是调度器的上限，不是你业务的上限。

## 什么时候不该用池

- **任务比一次 channel 发送还短。** 把 100ns 的闭包丢进池，同步开销比工作本身大。这种该批量，或直接在调用方做。
- **只是偶尔扇出几个 I/O。** `errgroup` + `SetLimit` 更短、生命周期更清楚，别在每个 handler 里持有一个进程级池再套一层。
- **任务会再向同一个池提交并等待。** worker 跑 `Submit` 然后阻塞等结果，队列一满就自死锁：所有 worker 在等空位，空位要等 worker 做完。同池禁止同步重入；跨池也要保证依赖是 DAG。
- **用池「加速」纯 CPU 且 worker 远大于 `GOMAXPROCS`。** 跑得更慢，调度器更忙。
- **在库里偷偷起一个全局池。** 调用方看不见并发度，两个库各起 256 个 worker，进程级策略被架空。池应当在应用入口显式构造、显式 `Close`。

还有一个和 GMP 相关的误区：池**不能**让你的 Go 代码跑过 `GOMAXPROCS` 个 P。它只能让「睡着的 G」变多或变少。若 `pprof` 显示 `runnable` 堆积，先看是不是 CPU 饱和，而不是先把 worker 再翻一倍。

## 和现成库怎么选

- 一批请求内的扇出：`errgroup`，`SetLimit`。
- 全进程共享的异步队列、需要拒绝策略和动态扩缩：看 [ants](https://github.com/panjf2000/ants)，不要复制它的热路径微优化。
- 自己写：固定 worker + 有界队列 + `TrySubmit` + panic 恢复 + `Close/Wait`，通常已经够。缺的那 90% 是 metrics（队列长度、等待时间、panic 次数）和文档化的拒绝语义。

读 ants 源码时可以对着本文的三列：它的 `Pool` 是动态 worker；`Invoke` / `Submit` 对应入队；`release` 是空闲回收。热路径上的 `sync.Pool`、spinlock 是规模到了之后的事，先把语义做对。

## 收束

Go 不需要线程池来摊掉 `clone` 的成本。它需要任务池来把「同时做多少件事」写成一个数字：超过的部分要么排队，要么拒绝，要么把反压传回 `ctx`。信号量限制的是在飞的 G；固定 worker 限制的是长期活着的 G 和队列；动态 worker 让这个数字跟负载走。

默认选择很窄：一次扇出用 `errgroup.SetLimit`；进程级流水线用有界队列的固定池；只有脉冲极明显时才上动态扩缩。panic 必须恢复，关闭必须先停提交再抽干，正在执行的任务只能靠 `context` 合作退出。这些比 worker 数量是 8 还是 16 更决定这个池会不会在线上把你锁死。
