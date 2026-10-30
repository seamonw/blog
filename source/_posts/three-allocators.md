---
title: 三种用户态分配器：ptmalloc、tcmalloc 与 jemalloc 的缓存和还页
date: 2026-10-30 15:00:00
categories: Cpp
tags:
	- 内存分配
	- ptmalloc
	- tcmalloc
	- jemalloc
	- golang
---

`free` 返回之后 RSS 不降，多半不是泄漏。小块先留在线程缓存里，页要等衰减或堆顶裁剪才还给内核。三种分配器的差别是缓存挂在谁身上、锁在哪一层、还页等不等。

用户态分配器从 `brk` / `mmap` 拿大块内存，再切成对象。请求会向上取整到 size class，多出来的字节是内部碎片。数字以 glibc 2.40、jemalloc 手册里的默认值，以及 Go 1.27 的 `runtime/malloc.go` 为准。

## 热路径不拿锁，未命中才进共享结构

小对象的快路径都是一块线程或 CPU 私有的空闲链表。命中就返回，不加锁。链表空了，才向带锁的中心结构要一整批，再从操作系统要页。

```text
malloc(n)
  |
  +-- 本地缓存有这个 size class --> 弹出一个，返回
  |
  +-- 没有 --> 中心空闲链表 / arena（这里才锁或做原子批次）
                 |
                 +-- 中心也没有 --> 页堆、arena 的 top，或 mmap
```

| | 本地缓存 | 共享层 | 数量上限 |
| --- | --- | --- | --- |
| ptmalloc（glibc 2.40） | 每线程 tcache。64 个 bin，每 bin 默认 7 个 chunk，最大 chunk 1032 字节 | arena 互斥锁。主 arena 走 `brk`，其余走 `mmap` | 超过 `arena_test`（默认 8）之后，自动创建的上限是 `8 * 核数` |
| tcmalloc | 有 rseq 时是 per-CPU cache；否则是 thread cache | central free list，再下面是 page heap | 缓存按 CPU 或线程数涨，不按核数封顶 |
| jemalloc | 每线程 tcache，默认开，最大档 32 KiB | arena。默认最多 `4 * 核数`（单核机器是 1） | 8 MiB 以上走单独的 huge arena |

ptmalloc 在 tcache 未命中之后还有 fastbin、small bin、large bin。64 位上 fastbin 默认只接请求不超过 128 字节的块，而且不和相邻空闲块合并。tcache 已经接住了这些小块的大部分。说「ptmalloc 每次分配都抢一把全局锁」只对 glibc 2.26 以前成立。2.26 起命中 tcache 不加锁。线程数超过 arena 上限时，抢的是某一把 arena 锁：`trylock` 失败就试下一个，都忙就等待。

tcmalloc 这个名字对应的旧结构是 thread cache、central free list、page heap。现在的 Google tcmalloc 在支持重启序列（rseq）的机器上，热路径改到 per-CPU cache，线程迁移时不会把缓存带错 CPU。没有 rseq 才退回 thread cache。大对象跳过这两级，直接找 page heap。

jemalloc 的 arena 默认按 `4 * 核数` 封顶，线程和 arena 多对多。`percpu_arena` 默认关闭，不要把它说成默认按 CPU 绑死。

Go 的分配器注释写明脱胎于 tcmalloc，但缓存挂在 P 上，不挂在线程或 goroutine 上。小对象上限 32 KiB，共 67 个可用 size class；无指针且小于 16 字节的对象塞进 16 字节的 tiny 块。P 的个数是 `GOMAXPROCS`，不是 goroutine 数。P 是什么见 [Go 调度器：G、M、P 与 goroutine 调度](/blog/2026/09/02/golang-gmp/)。

## free 把块放回缓存，不把页还给内核

`free` 的小块进入 tcache、fastbin 或 jemalloc 的 tcache。地址空间还在，RSS 还在。

```text
free(小对象)
  ptmalloc   进本线程 tcache；满了进 arena 的 fastbin / unsorted
             只有 mmap 出去的大块会 munmap
             堆顶连续空闲超过 trim 阈值（默认 128 KiB）才把 top 裁掉
  jemalloc   页标成 dirty。默认约 10 秒按衰减曲线变成 muzzy（MADV_FREE）
             dirty_decay_ms = 0 才是立刻 purge；-1 是永不 purge
  Go         槽位还进 mspan。整段 span 都空了才还 mheap
             还给操作系统是 scavenger 的事，不发生在 free 上
```

glibc 的 `mmap` 阈值默认 128 KiB，并且会按分配习惯往上调。64 位上的上限是 `4 MiB * sizeof(long)`，Linux 上 `long` 是 8 字节，也就是 32 MiB。512 KiB 是 32 位的上限。一旦用 `mallopt` 或环境变量把阈值设死，动态调整就关了。阈值以下的块 `free` 之后留在堆里，是设计，不是没走到 `munmap`。

jemalloc 还有一档：`muzzy_decay_ms` 默认 0，dirty 衰减成 muzzy 之后会马上再 purge。10 秒是 dirty 那一档的时间，不是「free 之后再等 10 秒 RSS 一定下降」。衰减是一批页的曲线，中间 RSS 会台阶式掉，不会在第 10 秒整突然归零。

## 多出来的字节是档位，不是泄漏

三种分配器各自的档位不同，取整方式一样：请求落进不小于它的那一档。下面用 Go 1.27 `SizeClassToSize` 的前 10 档做这件事。33 字节进 48 字节档，15 字节是内部碎片。1 字节进 8 字节档，浪费 7 字节，也就是这一档注释里的 87.5%。

```go
package main

import "fmt"

// 摘自 Go 1.27 internal/runtime/gc.SizeClassToSize 的前 10 档。下标 0 不用。
var small = []int{0, 8, 16, 24, 32, 48, 64, 80, 96, 112, 128}

func roundSmall(n int) int {
	for i := 1; i < len(small); i++ {
		if n <= small[i] {
			return small[i]
		}
	}
	return -1
}

func main() {
	for _, n := range []int{1, 9, 33, 128} {
		got := roundSmall(n)
		fmt.Printf("%d -> %d  waste %d\n", n, got, got-n)
	}
}
```

输出是 `1 -> 8 waste 7`、`9 -> 16 waste 7`、`33 -> 48 waste 15`、`128 -> 128 waste 0`。同一张表里 1025 字节进入 1152 字节档。ptmalloc 和 jemalloc 的档位边界不同，不能拿这四行去对 `malloc(33)` 的 chunk 大小。要对的是「请求大小」和「档位大小」的差，以及这个差乘以对象个数。

## 避坑

- **用 free 之后立刻采样的 RSS 判断泄漏。** ptmalloc 的小块要等线程缓存满、堆顶够长才会动；jemalloc 的 dirty 页默认按约 10 秒衰减。改法：看分配器自己的统计（glibc 的 `malloc_info`，jemalloc 的 `stats.resident` 和 `stats.mapped`），和进程 RSS 分开读。RSS 不动、统计里的空闲页在涨，是还页策略，不是没 free。
- **空闲线程把缓存钉住。** 每个线程的 tcache 别的线程用不了。线程池里的空闲线程会一直占着最多 `64 * 7` 个 glibc tcache chunk。改法：先数线程数再算缓存上限；确认泄漏前把线程退掉，或关掉 tcache 做一次对照。
- **按结构体 `sizeof` 估算堆。** 33 字节的对象在 Go 里占 48。再乘上每档 span 末尾对不齐的尾浪费（Go 的 24 字节档一个 8 KiB span 切出 341 个，尾上还剩 8 字节）。容量按档位和 span 算，不按 `sizeof`。

## 面试重点

| 追问 | 答到这里可以停 |
| --- | --- |
| 三者快路径 | 都是线程或 CPU 本地空闲链表，命中不加锁 |
| ptmalloc 的锁 | tcache 未命中才进 arena。64 位超过 8 个 arena 之后，上限是 8 倍核数 |
| tcmalloc 现在的热路径 | 有 rseq 时是 per-CPU cache，不是「每个线程一定有一份」 |
| jemalloc 为何 free 后 RSS 还在 | dirty 页默认约 10 秒衰减。0 是立刻 purge，-1 是关闭 |
| Go 的缓存挂在哪 | P 上的 mcache。个数等于 `GOMAXPROCS`，小对象上限 32 KiB |

### fastbin 不合并

tcache 满了之后，128 字节以内的块进 fastbin。fastbin 是单链表，块头的 inuse 位仍保持置位，相邻的空闲块不会合成一块。两个 64 字节空闲块并排，也满足不了 96 字节的请求。合并要等 `malloc_consolidate`：一次不小于 64 KiB 的 `free`（`FASTBIN_CONSOLIDATION_THRESHOLD`）会把 fastbin 倒进 unsorted bin 再合并。只分配小对象的进程，碎片来自这里，不来自 mmap。

### 缓存个数不等于线程数，也不等于核数

ptmalloc 的 tcache 跟线程走，arena 跟 `8 * 核数` 走，两套数字。jemalloc 的 tcache 跟线程走，arena 默认跟 `4 * 核数` 走。Go 的 mcache 只跟 P 走。把「无锁」理解成「每个 goroutine 一份缓存」，会把内存上限算成 goroutine 数量乘以档位数。P 复用同一份 mcache，goroutine 切换出去再回来，看到的是这个 P 上的缓存，不是自己独占的那一份。

## 总结

三种分配器都用 size class 换少几次系统调用，热路径都是本地缓存。ptmalloc 的共享层是有上限的 arena 锁，tcmalloc 的热路径在支持 rseq 时按 CPU，jemalloc 把空闲页留在 dirty 上按秒衰减。`free` 返回只说明块回到了缓存或空闲链表。页还没还，RSS 就不会动。

## 参考

- glibc 2.40 `malloc/malloc.c`：`NARENAS_FROM_NCORES`、`TCACHE_FILL_COUNT`、`DEFAULT_MMAP_THRESHOLD`
- jemalloc 手册：`opt.narenas`、`opt.dirty_decay_ms`、`opt.tcache_max`
- Go 1.27 `src/runtime/malloc.go`、`src/internal/runtime/gc/sizeclasses.go`
