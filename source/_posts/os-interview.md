---
title: 操作系统面试全解：虚拟内存、进程线程、Page Cache 与零拷贝
date: 2026-10-30 17:15:00
categories: 系统
tags:
	- 操作系统
	- linux
	- 虚拟内存
	- 零拷贝
	- 面试
---

只有 4 GB 物理内存的机器，`malloc(8 GB)` 却能成功返回；进程调完 `write()` 写文件立刻被 `kill -9` 杀掉，数据却一字节没丢；用 `rm` 删掉了 50 GB 的日志，`df -h` 显示磁盘依然 100% 满。这三个现象分别对应操作系统的三条底层分界：`malloc` 只给虚拟地址、`write` 只写到内核 Page Cache、`rm` 只减 `inode` 的硬链接计数。

本文按小林 coding《图解系统》的主干，把资深后台面试必考的**虚拟内存与回收、进程线程与切换、IPC 与并发锁、文件系统与 Page Cache、零拷贝与高性能 IO 模型**收拢成一篇。

---

## 一、虚拟内存、缺页中断与内存回收

### 1. 多级页表、TLB 与两类缺页中断

每个进程独占一套虚拟地址空间，通过页表映射到物理页（4 KB）。64 位系统（常用低 48 位地址）用**四级页表**按需分配页表节点，避免单级页表提前占满内存。

```text
CPU 发出虚拟地址
  |
  +--> 查 TLB（快表，硬件缓存近期页表项）
         |-- 命中 -----------------------+
         |-- 未命中：由 MMU 硬件遍历内存中的四级页表（Page Walk）
                       |-- 页表项有效 --> 填入 TLB --> 拼出物理地址访问内存
                       |-- 页表项无效（未映射或已换出）
                             |
                             v
                       触发缺页异常（Page Fault，陷入内核态）
                         |-- 软缺页（Minor）：物理页已在内存（如共享库、其它进程已加载），只建映射，不读盘
                         |-- 硬缺页（Major）：物理页不在内存，分配物理页并从磁盘/Swap 读入，慢几个数量级
```

**判据**：**TLB 未命中 $\neq$ 缺页异常**。TLB 未命中只是多走一次内存里的页表查询（几十纳秒）；只有页表项本身还没映射物理页（`Present = 0`）时，才会触发缺页异常进内核。

### 2. 在 4 GB 物理内存的机器上申请 8 GB 内存会发生什么？

`malloc()` 申请的只是**虚拟内存（`vm_area_struct`）**，只要不读写，内核根本不分配物理页。

| 环境与阶段 | 结果与内核行为 |
| --- | --- |
| **32 位系统** | 用户态虚拟空间上限仅 3 GB，`malloc(8 GB)` **直接返回 `NULL`** |
| **64 位系统，仅 `malloc(8 GB)` 不访问** | 虚拟空间高达 128 TB。若 `vm.overcommit_memory` 为 `0`（默认启发式，单次申请超物理内存+Swap 会拒绝）或 `2`（严禁超额），`malloc` 失败；若设为 `1`（允许无条件超额），**`malloc` 直接成功** |
| **64 位系统，申请成功后逐页写入 8 GB** | 每次写新页触发缺页中断分配物理页。物理内存不够时：<br>1. **开启了足够大的 Swap（如 $\ge$ 4 GB）**：冷页被换出到磁盘，**程序正常运行但变慢**；<br>2. **未开启 Swap 或 Swap 也不足**：直接内存回收失败，触发 **OOM Killer** 杀掉得分最高的进程 |

用户态分配器（`brk` 与 `mmap` 阈值、`tcache` 与 `arena` 锁）的内部细节，见 [三种用户态分配器：ptmalloc、tcmalloc 与 jemalloc 的缓存和还页](/blog/2026/10/30/three-allocators/)。

### 3. 物理内存紧张时的三级反映

```text
空闲内存充足 ----> 直接分配
       |
       v 跌破后台水线（Watermark Low）
唤醒 kswapd 内核线程异步回收（回收干净的 Page Cache，或将匿名页 swap out）
       |
       v 分配速度过快，跌破最低水线（Watermark Min）
触发直接内存回收（Direct Reclaim）：阻塞当前申请内存的业务线程，同步回收（造成延迟毛刺）
       |
       v 回收后仍拿不到足够的物理页
触发 OOM Killer：按 oom_score（占用物理页越多得分越高）选进程发 SIGKILL 杀掉
```

### 4. 如何同时解决「预读失效」与「缓存污染」？

Linux 的 Page Cache 与 MySQL InnoDB Buffer Pool 都不用朴素的单链表 LRU：
- **预读失效**：磁盘顺序读会提前把后面几页读进缓存；若这几页根本没被访问却插在普通 LRU 头部，反而把真正的热点页挤出去了。**解法：冷热分层**（Linux 分 `active_list` 和 `inactive_list`；MySQL 分 `young` 和 `old` 区），**预读页只进冷区头部**，没被访问就直接从冷区淘汰，碰不到热区。
- **缓存污染**：一次全表扫描或 `grep` 大文件，所有页都被访问了 **1 次**。若访问 1 次就升入热区，热区依然会被洗劫一空。**解法：提高进入热区的门槛**——Linux 要求页在 `inactive_list` 中被**访问第 2 次**才升入 `active_list`（MySQL 还要加判在 `old` 区停留满 1 秒）。

下面这段可运行的 Go 代码实现了双链表二次访问晋升模型：

```go
package main

import (
	"fmt"
	"slices"
)

type TwoListLRU struct {
	active   []int
	inactive []int
	seen     map[int]int
	cap      int
}

func NewTwoListLRU(cap int) *TwoListLRU {
	return &TwoListLRU{cap: cap, seen: make(map[int]int)}
}

// 预读只进 inactive 头部，绝不挤占 active 热点
func (c *TwoListLRU) Readahead(page int) {
	if slices.Contains(c.active, page) || slices.Contains(c.inactive, page) {
		return
	}
	if len(c.inactive) >= c.cap {
		evict := c.inactive[len(c.inactive)-1]
		c.inactive = c.inactive[:len(c.inactive)-1]
		delete(c.seen, evict)
	}
	c.inactive = append([]int{page}, c.inactive...)
}

// 真正访问满 2 次，才从 inactive 升入 active
func (c *TwoListLRU) Access(page int) {
	c.Readahead(page)
	c.seen[page]++
	if c.seen[page] < 2 || slices.Contains(c.active, page) {
		return
	}
	c.inactive = slices.DeleteFunc(c.inactive, func(p int) bool { return p == page })
	if len(c.active) >= c.cap {
		demote := c.active[len(c.active)-1]
		c.active = c.active[:len(c.active)-1]
		c.seen[demote] = 1
		c.Readahead(demote)
	}
	c.active = append([]int{page}, c.active...)
}

func main() {
	lru := NewTwoListLRU(2)
	for _, p := range []int{1, 1, 2, 2} {
		lru.Access(p) // 页 1、2 访问两次进入 active
	}
	lru.Readahead(3) // 预读未访问的页 3、4
	lru.Readahead(4)
	lru.Access(5) // 批量扫描只访问一次的页 5、6
	lru.Access(6)

	fmt.Println("active:", lru.active)     // [2 1] 热点保留
	fmt.Println("inactive:", lru.inactive) // [6 5] 一次性页留在冷区
}
```

---

## 二、进程、线程、切换开销与崩溃信号

### 1. 为什么线程切换比进程切换快得多？

进程是**资源分配**的基本单位（拥有独立的虚拟地址空间、页表、打开文件表）；线程是 **CPU 调度**的基本单位（共享地址空间与堆，仅独享栈、寄存器、`PC`）。

| 切换动作 | 同进程内的线程切换 | 跨进程切换 |
| --- | --- | --- |
| 保存/恢复内核栈与 CPU 寄存器 | 需要 | 需要 |
| **切换页表基址寄存器（`CR3`）** | **不需要**（共享同一套页表） | **必须切换** |
| **TLB 快表与硬件缓存影响** | **TLB 继续有效**，Cache 命中率高 | **`CR3` 一换，TLB 全部失效冲洗**，随后密集触发 Page Walk |

**结论**：线程切换快，不是因为少保存了几个寄存器，而是**不用切页表、不会把 TLB 刷空**。若切到用户态协程，连进出内核态都省了，见 [Go 调度器：G、M、P 与 goroutine 调度](/blog/2026/09/02/golang-gmp/)。

### 2. `fork()` 写时复制（COW）与多线程 `fork` 死锁

- **写时复制（Copy-on-Write）**：`fork()` 时内核**不复制物理内存页**，只把父进程的页表复制一份给子进程，并将双方页表项全部标为**只读**。任一方试图写入时，CPU 触发写保护缺页异常，内核这才分配一页新的物理内存、拷入数据并改回可写。
- **为什么在多线程程序里调用 `fork()` 极易死锁？**
  `fork()` **只会把当前调用它的那 1 条线程复制到子进程**，父进程里的其它线程在子进程中瞬间蒸发。如果蒸发掉的某个线程当时正持有一把互斥锁（比如 `malloc` 内部的 `arena` 锁或日志锁），这把锁在子进程里会被**永远锁死且无人释放**；子进程一旦调 `malloc` 或 `printf` 就会立刻死锁。所以在多线程进程中，`fork()` 后除了立刻 `exec()` 之外，不要做任何会拿锁的操作。

### 3. 线程崩溃了，进程一定会崩溃吗？

- **C/C++ 默认情况：会。** 线程写只读内存、解引用空指针或越界访问内核地址时，是因为共享同一个虚拟地址空间。MMU 捕获非法访问后，内核向**整个进程**发送 `SIGSEGV`（段错误）信号，默认信号处理函数直接终止整个进程。
- **为什么 Java/Go 里单个线程空指针或栈溢出，进程却没崩？**
  因为 JVM / Go Runtime 在启动时**注册了自定义的 `SIGSEGV` 信号处理函数**。当某个线程触发 `SIGSEGV` 时，内核把控制权交给运行时的信号处理函数；运行时检查故障地址，发现是用户栈保护页或 `0x0` 空指针，就在该线程栈上改写恢复点，将其转换成可捕获的 `NullPointerException` / `StackOverflowError` 或 Go `panic`，而不让进程退出。

---

## 三、进程间通信（IPC）、锁与死锁

### 1. IPC 方式对比：为什么共享内存最快？

| IPC 方式 | 数据拷贝与内核参与 | 特点与边界 |
| --- | --- | --- |
| **匿名管道 / 命名管道（FIFO）** | 用户态 -> 内核缓冲区 -> 对端用户态（**2 次拷贝，2 次切换**） | 字节流、半双工、自带阻塞同步；匿名管道仅限父子/兄弟进程 |
| **消息队列** | 同样经过内核链表中转（**2 次拷贝**） | 保留消息边界与类型，但不适合传大块数据 |
| **信号（Signal）** | 异步事件通知，几乎无数据载荷 | 唯一的**异步**通信方式（如 `SIGKILL`、`SIGTERM`） |
| **信号量（Semaphore）** | 内核维护的原子计数器（P/V 操作） | 不传数据，专门给多进程做互斥与同步（常配合共享内存） |
| **共享内存（Shared Memory）** | 两个进程的虚拟地址映射到**同一块物理内存页**（建立后 **0 次内核拷贝、0 次系统调用**） | **最快**，但本身没有任何同步保护，必须配信号量或互斥锁 |
| **Socket（含 Unix Domain Socket）** | 走内核 sockbuf（本机 UDS 绕过网络协议栈） | 支持跨机器；本机进程间高吞吐首选 UDS |

### 2. 条件变量的两个铁律、自旋锁与死锁

- **自旋锁 vs 互斥锁**：自旋锁拿不到锁时在 CPU 上忙等（不切线程），适合**持锁时间短于两次线程上下文切换开销**的场景（单核 CPU 上用自旋锁且不开抢占等于死锁）；互斥锁拿不到就让出 CPU 睡眠。
- **为什么条件变量 `cond_wait` 必须配合互斥锁？**
  防止「先检查条件不满足、准备去睡，但在睡下去之前另一线程已经修改条件并 `signal`」的 **Lost Wakeup（丢失唤醒）竞态**。把「检查条件」和「挂入等待队列」包在同一把互斥锁里，`cond_wait` 内部原子地完成「解锁 + 睡眠」，被唤醒返回前再自动重新拿锁。
- **为什么等待条件必须写 `while (!ready)` 而不能写 `if (!ready)`？**
  因为存在**虚假唤醒（Spurious Wakeup）**（多核信号中断或广播唤醒多个线程，等当前线程抢到锁时，条件可能已经被别的线程重新消费掉了），醒来后必须再验一次条件。
- **死锁四个必要条件**：互斥、持有并等待、不可剥夺、环路等待。工程上最实用的破除法是破坏第 4 条——**所有线程严格按统一顺序加锁**（例如按锁对象的地址或账户 ID 升序拿锁）。多核下无锁编程与 CPU 缓存伪共享（64B Cache Line）见 [C++ memory order：多线程下的可见性与同步](/blog/2026/08/28/cpp-memory-order/)。

---

## 四、文件系统、Page Cache 与落盘边界

```text
进程 fd 表 ---> struct file (当前读写偏移) ---> inode (权限、大小、硬链接数 i_nlink、数据块指针)
                                                 |
                                                 v
                                       Page Cache (4KB 内存缓存页)
                                                 |  fsync / 后台 flusher 线程
                                                 v
                                            磁盘数据块
```

1. **硬链接 vs 软链接**：硬链接是多个目录项直接指向**同一个 `inode`**（`i_nlink++`，不可跨文件系统，不可链目录）；软链接是一个拥有独立 `inode` 的新文件，里面存的是目标文件的**路径字符串**（目标删了就悬空）。
2. **文件已经被 `rm` 删了，为什么 `df` 看磁盘空间一点没释放？**
   一个文件真正从磁盘释放，必须同时满足 **`i_nlink == 0`（没有目录项指向它）** 且 **`i_count == 0`（没有任何进程还打开着它的 `fd`）**。线上进程打开着 `app.log` 持续写，你直接 `rm app.log` 只是把 `i_nlink` 减为 0，进程手里的 `fd` 仍握着 `i_count > 0`，数据块根本不会释放。**解法**：用 `lsof | grep deleted` 找到进程重启，或平时用 `> app.log`（截断文件）代替 `rm`。
3. **进程调用 `write()` 写文件时突然 `kill -9` 崩溃，已写数据会丢吗？**
   **不会丢。** 缓冲 IO 下，`write()` 返回成功代表数据已经从用户态拷进了**内核的 Page Cache**（标为脏页）。进程虽然被杀死了，但**操作系统内核没崩**，内核的后台回写线程照样会把 Page Cache 里的脏页刷回磁盘。只有**整机断电或内核 Panic**，才会丢掉还在 Page Cache 里未落盘的数据。
4. **`fsync` vs `fdatasync`**：每次写完立刻保证断电不丢，必须调 `fsync(fd)`（同时刷文件数据 + `inode` 元数据，需 2 次磁盘 IO）或 `fdatasync(fd)`（只刷数据与必要的大小变化，跳过修改时间等元数据，少 1 次磁盘 IO，数据库 WAL 日志首选）。

---

## 五、零拷贝、网络 IO 与中断

### 1. 硬中断、软中断与工作队列

网卡收包不能在关中断的硬中断里做完，否则会卡死其它硬件响应，因此拆成两半：
- **上半部（硬中断）**：CPU 响应网卡中断，只把网卡 RingBuffer 描述符摘下、屏蔽硬中断，随后触发**软中断标记**，耗时微秒级。
- **下半部（软中断 `NET_RX_SOFTIRQ` / `ksoftirqd`）**：开中断运行，通过 NAPI 轮询从 RingBuffer 批量把 `sk_buff` 取出走 TCP/IP 协议栈，放入 socket 接收队列并唤醒 `epoll`。
- **工作队列（Workqueue）**：硬中断和软中断都处于中断上下文，**绝对不能睡眠**；需要睡眠阻塞的慢操作才交给具有内核线程上下文的工作队列。

### 2. 传统文件传输 vs 三种零拷贝对比

把磁盘上的文件原封不动发给网卡，传统 `read() + write()` 经历 **4 次上下文切换 + 2 次 DMA 拷贝 + 2 次 CPU 拷贝**：

```text
传统 read + write (4次切换, 2次CPU拷贝, 2次DMA拷贝):
磁盘 --DMA--> [内核 Page Cache] --CPU拷贝--> [用户缓冲区] --CPU拷贝--> [内核 Socket 缓冲] --DMA--> 网卡

sendfile + SG-DMA 网卡支持 (2次切换, 0次CPU拷贝, 2次DMA拷贝):
磁盘 --DMA--> [内核 Page Cache] ----(只把物理页地址与长度描述符写入 Socket 缓冲)----> SG-DMA 直接从 Page Cache 读入网卡
```

| 传输方式 | 系统调用 / 上下文切换 | CPU 拷贝次数 | DMA 拷贝次数 | 适用场景与局限 |
| --- | --- | --- | --- | --- |
| 传统 `read + write` | 2 次调用 / **4 次切换** | **2 次** | 2 次 | 需要在用户态修改、加密或压缩数据 |
| `mmap + write` | 2 次调用 / **4 次切换** | **1 次**（Page Cache -> Socket） | 2 次 | 省掉用户态缓冲内存，但映射小文件有页表与 TLB 开销 |
| `sendfile`（支持 SG-DMA） | 1 次调用 / **2 次切换** | **0 次** | 2 次 | 静态小文件高速转发（如 Kafka 消费、Nginx 静态资源） |
| **异步 IO + Direct IO** | 异步提交 | **0 次**（绕过 Page Cache） | 2 次 | **超大文件传输**或自管缓存的数据库（MySQL） |

**为什么传输 GB 级超大文件时，Nginx 反而不推荐用 `sendfile`？**
因为 `sendfile` 强依赖 **Page Cache**。传输几个 GB 的大文件会把内存里的 Page Cache 全部冲刷占满，把其它热点小文件全部挤出去（造成严重的**缓存污染**），而且大文件读完就走、不会复用，进出 Page Cache 纯属白费功夫。因此超过阈值的大文件应关闭 `sendfile`，改用 **异步 IO + Direct IO（`O_DIRECT`）** 直接绕过 Page Cache。

### 3. Reactor vs Proactor

- **Reactor（同步非阻塞，Linux 主流）**：应用向 `epoll` 注册感兴趣事件；**内核负责感知「数据到了（可读）」**并通知应用，**应用线程自己调非阻塞 `read()` 把数据从内核拷到用户态**。单 Reactor 单线程（Redis）、主从 Reactor 多线程（Netty / Nginx）都属此类。为什么 Reactor 里的 `read` 必须是非阻塞的，见 [为什么 IO 多路复用必须搭配非阻塞 IO](/blog/2026/09/16/io-multiplexing-nonblocking/)。
- **Proactor（异步 IO）**：应用直接把用户态 `buf` 交给内核发起异步读；**内核负责把数据从网卡/磁盘一直拷进用户态 `buf`，全部干完后通知应用「读操作已完成」**。Windows `IOCP` 与 Linux 5.1+ 的 **`io_uring`** 才是真正的 Proactor。

---

## 六、高频考点速查表

| 面试追问 | 一句话判据与标准答案 |
| --- | --- |
| 用户态切换到内核态有哪 3 种入口？ | **系统调用**（主动）、**异常**（如缺页异常、除零，被动同步）、**外设硬中断**（被动异步） |
| 僵尸进程 vs 孤儿进程？ | 子进程退出了但父进程没调 `wait/waitpid` 回收其进程控制块，是**僵尸进程**（占 PID，杀父进程让 `init` 收养才能清）；父进程先死了，子进程被 `init(PID 1)` 收养，是**孤儿进程**（无害） |
| 一个进程最多能创建多少个线程？ | 64 位下虚拟内存充足，主要受三个内核参数限制：每线程栈大小（`ulimit -s`，默认 8MB）、系统最大线程数 `/proc/sys/kernel/threads-max`、最大进程/线程 PID 上限 `pid_max` 及最大内存映射区数量 `vm.max_map_count` |
| `select`、`poll`、`epoll` 的本质区别？ | `select/poll` 每次调用都要把全部 fd 集合拷进内核并 $O(N)$ 线性遍历；`epoll` 用内核**红黑树**存监控的 fd（增删 $O(\log N)$），用设备驱动回调把就绪 fd 挂入**双向就绪链表**，`epoll_wait` 只取就绪链表 $O(1)$ |
| `mmap` 读文件一定比 `read` 快吗？ | 不一定。小文件或单次顺序读时，`mmap` 建立虚拟内存映射、触发缺页中断和卸载映射的页表/TLB 锁开销，往往比直接 `read` 一次拷贝还慢；`mmap` 赢在**多进程共享只读**或**频繁随机原地读写** |

## 总结

操作系统面试的核心只有三条线：**地址翻译线**（虚拟地址靠页表和 TLB 转物理页，`malloc` 不占物理内存，线程切换赢在不刷 TLB）、**缓存与落盘线**（`write` 只进 Page Cache，双链表二次访问防预读失效与缓存污染，断电靠 `fsync`）、**数据搬运线**（`epoll` 解决等谁，零拷贝靠 `sendfile` + SG-DMA 消掉 CPU 拷贝，超大文件用 Direct IO 绕开 Page Cache）。
