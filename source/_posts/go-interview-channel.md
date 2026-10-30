---
title: Go 面试：channel 的阻塞、关闭与 select
date: 2026-10-27 21:30:00
categories: Golang
tags:
	- golang
	- 面试
	- channel
---

channel 的面试很少死在「有没有缓冲」，死在关闭之后谁还在发、缓冲里剩下的值谁来读、`select` 里一个 nil channel 会怎样。把这三件事的表背下来，剩下的是这条路径怎么挂起 goroutine。

同系列：[Go 面试：slice 的头部、扩容与共享底层数组](/blog/2026/10/27/go-interview-slice/)、[Go 面试：map 的 Swiss Table、扩容与并发](/blog/2026/10/27/go-interview-map/)。下面的发送、接收、关闭顺序以 Go 1.27 的 `runtime/chan.go` 为准。阻塞时 G 怎么让出线程，见 [Go 调度器：G、M、P 与 goroutine 调度](/blog/2026/09/02/golang-gmp/)。

## 一个 hchan，三条路

```text
recvq : sudog -> sudog -> ...     等着收的 G
buf   : [ a | b | _ | _ ]         环形，sendx / recvx，仅 dataqsiz > 0
sendq : sudog -> sudog -> ...     等着发的 G
closed, lock
```

`chansend` 在锁里按这个顺序走：

1. 已经关闭：解锁，`panic("send on closed channel")`。
2. `recvq` 上有人：值直接拷进接收方的栈，不进缓冲，叫醒对方。
3. 缓冲还有空位：写入 `sendx`，`qcount++`。
4. 否则把当前 G 做成 `sudog` 挂进 `sendq`，`gopark`。

接收对称：有发送者等着，就从发送者手里拿，缓冲里有就从 `recvx` 拿，否则挂进 `recvq`。无缓冲 channel 的 `buf` 是 nil，第 2 步是它唯一的成功路径，不存在「先放进长度为 0 的队列」。

关闭把 `closed` 置 1，然后把 `recvq` 和 `sendq` 全部摘下来叫醒。接收方被叫醒时元素清成零值，`ok` 为 false。发送方被叫醒后看到 `success == false`，同样 `panic("send on closed channel")`。`close(nil)` 和关第二次都在锁里直接 panic。

所以同一条 channel 上，操作和状态是这张表：

| | nil | 开着 | 已关闭 |
| --- | --- | --- | --- |
| 发送 | 永远阻塞 | 交给等待的接收者，或写入缓冲；都没有就阻塞 | panic |
| 接收 | 永远阻塞 | 从等待的发送者或缓冲取值；都没有就阻塞 | 先读完缓冲，之后是零值，`ok == false` |
| `close` | panic | 唤醒所有等待者 | panic |

缓冲里还剩两个值时关闭，接收顺序是「剩下的值，然后再是零值」。Go 1.27 上 `make(chan int, 2)` 放入 1、2 再 `close`，四次接收得到 `1, true`、`2, true`、`0, false`、`0, false`。`for range` 会在 `ok` 变 false 时停，所以能读到 1 和 2，不会读到那个零值。没人 `close` 的话，`range` 就停不下来。

## 谁关闭，select 选哪一路

关闭是「没有更多的值了」，不是「把通道销毁」。发送方关掉它，接收方用 `range` 或 `ok` 收尾。多个发送方谁都不该直接 `close`，除非另有一个协调者用 `WaitGroup` 或计数确认都写完了。

```go
package main

import "fmt"

func produce(out chan<- int) {
    defer close(out) // 唯一的发送方，离开时关闭
    for i := 1; i <= 3; i++ {
        out <- i
    }
}

func main() {
    ch := make(chan int)
    go produce(ch)
    for v := range ch {
        fmt.Println(v)
    }
}
```

`select` 里有多路能走时，伪随机挑一路，不是源码顺序，也不是谁先写谁优先。一路都走不了，有 `default` 就走 `default`；没有就挂起，直到某一路可以走。

nil channel 在 `select` 里永远不会被选中。这是故意留的开关：把一个 `case` 的 channel 换成 nil，这路就从选择里消失。

```go
func forward(in <-chan int, out chan<- int, done <-chan struct{}) {
    for {
        select {
        case v, ok := <-in:
            if !ok {
                return
            }
            out <- v
        case <-done:
            return
        }
    }
}
```

`done` 用 `close` 广播，所有听着它的接收同时返回。不要往 `done` 里送值来表示取消，多个等待者只有一个收得到。生产代码里这种取消更常写成 `context.Context`，机制上仍是「关掉一条大家都能看见的信号」，见 [Go context：设计、源码与代价](/blog/2026/09/02/golang-context/)。

内存模型用 synchronized before 写顺序，它计入 happens-before。和 channel 有关的几条，方向不要说反：

- 发送 synchronized before 对应接收的完成
- `close` synchronized before 一次因为关闭而返回零值的接收
- 无缓冲 channel 上，接收 synchronized before 对应发送的完成
- 容量为 C 时，第 k 次接收 synchronized before 第 k+C 次发送的完成

第三条只对无缓冲成立：发送返回时，对面的接收已经发生。第四条把这件事推广到缓冲，也是用缓冲 channel 做计数信号量时，那次「发送能返回」所依赖的顺序。

## 避坑

- **接收方关闭。** 发送方还在写就会 panic。channel 没有「关闭状态查询」这种安全接口，约定只能靠谁创建、谁发送来维持。
- **`default` 放在本该阻塞的地方。** `select` 加 `default` 之后这个调用不再等待，包进 `for` 就是忙等。要等就别写 `default`。
- **把 `ok` 丢掉。** `v := <-ch` 在关闭后一直给出零值，和「对端真的送了一个零值」分不清。要区分就用 `v, ok := <-ch`，或者 `for range`。
- **无缓冲当成容量 0 的队列。** 队列模型解释不了「值从发送方栈直接拷到接收方栈」。对面没有人在收，无缓冲发送就停住，不会先帮你存着。

## 面试重点

| 追问 | 答到这里可以停 |
| --- | --- |
| 无缓冲和有缓冲 | 无缓冲必须对端在等，值直接拷贝。有缓冲先看环形队列，满了才挂 `sendq` |
| 关闭之后还能读吗 | 能。先排空缓冲，然后零值且 `ok == false`。`range` 在这里停 |
| 关闭之后还能写吗 | 不能，panic。已经堵在 `sendq` 上的发送者被叫醒后也 panic |
| 谁来 close | 发送方。多个发送方要有单独的协调，接收方不关 |
| nil channel | 单独收发永久阻塞。`close(nil)` panic。在 `select` 里这路永不入选 |
| `select` 多路就绪 | 伪随机选一路，不按书写顺序 |
| 和调度的关系 | 等在 channel 上的是 G，挂在 `sudog` 上，不占着一个能跑的 P 空转 |

## 总结

channel 是带锁的队列，加上两条等待链表。发送、接收、关闭的组合就那一张表，表外面没有第四种状态。面试里先写表，再讲无缓冲为什么不进 `buf`，最后讲谁负责 `close`。顺序反了，很容易把 panic 和零值说反。
