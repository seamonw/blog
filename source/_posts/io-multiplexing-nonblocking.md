---
title: 为什么 IO 多路复用必须搭配非阻塞 IO
date: 2026-09-16 14:31:00
categories: 网络
tags:
	- io
	- epoll
	- nonblocking
	- linux
---

`epoll_wait` 返回了一个 `EPOLLIN`，你在这个 fd 上调 `read`。如果这个 fd 是阻塞的，这一次 `read` 可能让当前线程进入 `TASK_INTERRUPTIBLE` 睡下去——连带 `events[]` 数组里排队的另外九千九百九十九个连接一起停摆，直到某个对端愿意发一个字节，或者直到 TCP 超时。

这不是危言耸听，`select(2)` 的 BUGS 段落原文就写着：

> On Linux, `select()` may report a socket file descriptor as "ready for reading", while nevertheless a subsequent read blocks. ... Thus it may be safer to use `O_NONBLOCK` on sockets that should not block.

这句话解释了一个被反复混淆的分工：

- **IO 多路复用解决的是「在哪些 fd 上等」**：把 N 个 fd 的等待折叠成一次系统调用，避免一连接一线程。
- **非阻塞解决的是「等到之后，这一次调用不许睡」**：内核缓冲区没货就立刻 `EAGAIN` 返回，把控制权交回事件循环。

两件事在不同的维度上，不能互替。多路复用不会让 `read` 变得不睡，非阻塞不会帮你知道该读谁。少了后者，你写出来的是一个**单线程版的一连接一线程**——所有连接共享同一个可以被任意一个慢对端按住的执行流。

## 一、阻塞和非阻塞，差的是一个状态位

一次 `read(fd, buf, n)` 在内核里的路径大致是：拿 socket 锁，看 `sk_receive_queue` 有没有可拷的数据。有，就拷给你、返回字节数。没有，才出现分叉：

```text
              read() 进内核，sk_receive_queue 为空
                            │
        ┌───────────────────┴───────────────────┐
     阻塞 fd                                非阻塞 fd
        │                                       │
  加入 sk->sk_wq 等待队列                 立刻 return -EAGAIN
  set_current_state(TASK_INTERRUPTIBLE)         │
  schedule()  ← 让出 CPU，线程不可运行     调用者拿回控制权
        │
  被数据 / FIN / RST / 信号唤醒
```

`O_NONBLOCK` 这一位不改变数据从哪来、拷多少，它只改变**「无事可做时要不要 `schedule()`」**。所以它是唯一能保证「系统调用有界返回」的东西——超时、事件通知、多路复用都做不到这件事。

各个调用在两种模式下的语义对照：

| 调用 | 阻塞 fd（无事可做时） | 非阻塞 fd（无事可做时） |
| --- | --- | --- |
| `read` / `recv` | 睡到至少 1 字节可读 | `-1` / `EAGAIN`（或 `EWOULDBLOCK`） |
| `write` / `send` | 睡到 `sk_sndbuf` 腾出空间，再尽量拷 | 拷多少算多少，一个字节都拷不进才 `EAGAIN` |
| `accept` | 睡到 backlog 里有连接 | `-1` / `EAGAIN` |
| `connect` | 睡到三次握手完成或失败 | `-1` / `EINPROGRESS`，之后靠可写事件 + `SO_ERROR` 收尾 |

两个细节值得单独拎出来，后面的坑全从这里长出来：

- `EAGAIN` 和 `EWOULDBLOCK` 在 Linux/glibc 上是同一个值，但 POSIX 不要求，可移植代码两个都判。
- **写操作天然是「部分成功」的**。`write` 返回 `n < len` 不是错误，是内核告诉你发送缓冲区只剩这么多。不处理短写，就等于把消息截断。

## 二、多路复用只担保「此刻」，而你要的是「这一次调用」

`epoll_wait` 返回 fd 可读，含义精确地说是：**在内核把这个 fd 放进 ready list 的那一刻，它认为该 fd 上有事可做**。从那一刻到你真正发起 `read`，中间隔着用户态的一整段执行：遍历 `events[]`、查连接表、也许还有一次日志和一次锁。这段窗口里能出四类事，每一类都能让阻塞 fd 睡下去。

**路径一：内核本来就可能谎报。** 这是最反直觉的一类。`select(2)` BUGS 里举的例子是：数据到了，内核先把 socket 标成可读，后续校验和检查不通过，包被丢弃——可读状态兑现不了。man 页明说「There may be other circumstances in which a file descriptor is spuriously reported as ready」。历史上还有一个更明确的版本：`SO_RCVLOWAT` 在 Linux 2.6.28 之前不被 `select`/`poll`/`epoll` 尊重，只要有 1 个字节就报可读，而后续 `read` 会一直阻塞到 `SO_RCVLOWAT` 个字节齐了才返回。

**路径二：数据被别人先读走了。** 同一个 fd 被多个线程处理（同一个 epoll 实例被多线程 `epoll_wait`、两个 epoll 实例监听同一个 fd、`fork` 后共享 listen fd），事件可能投给 A，数据被 B 拿走。listen fd 上这件事有 man 页背书，`accept(2)` NOTES：

> There may not always be a connection waiting after ... `select(2)`, `poll(2)`, or `epoll(7)` return a readability event because the connection might have been removed by an asynchronous network error or another thread before `accept()` is called. If this happens, then the call will block waiting for the next connection to arrive.

**路径三：你自己读了第二次。** 这条最常见，也最容易被「我只 read 一次，不会阻塞」的自信掩盖。业务要的是一条完整消息，不是一次 `read` 的结果。于是代码长成这样：

```c
// 想凑齐 4 字节长度头 + body，于是循环
while (got < need) got += read(fd, buf + got, need - got);
```

第一次 `read` 消耗完内核里那 200 字节后，第二次进内核时队列已经空了。epoll 的那次唤醒早就兑现过了，它不会替这次调用兜底。**阻塞 fd 上，「读满」和「不睡」是互斥的两个要求。**

**路径四：可写不等于写得完。** `select(2)` 对 `writefds` 的定义后面直接跟了一句限定：

> However, even if a file descriptor indicates as writable, a large write may still block.

可写只意味着发送缓冲区有空间（Linux 上门限约为 `sk_sndbuf` 的一半左右，取决于协议和 `tcp_wmem`）。对端窗口收缩、你要写 1MB 而缓冲区只剩 8KB，阻塞 fd 上的 `write` 会在拷完 8KB 之后继续睡着等 ACK。

### 两张时序图

一万个连接，其中一个对端慢（或恶意），事件循环用阻塞 fd：

```text
t0   epoll_wait 返回 128 个就绪 fd，events[0] = fd 42
t1   handle(42): read → 200 字节
t2   解析：长度头说这条消息 4096 字节，还差 3896
t3   handle(42): read 再来一次 → 队列空 → TASK_INTERRUPTIBLE，schedule()
       ↓
     events[1..127] 一个都没被处理
     epoll_wait 也没人去调，新事件只在内核 ready list 里堆着
       ↓
t4   对端故意不发剩下的字节（或链路断了，等 RST/keepalive）
t5   ...几十秒后被唤醒
     整个进程的 P99 = 这个慢对端的心情
```

同一场景，fd 设了 `O_NONBLOCK`：

```text
t0   epoll_wait 返回 128 个就绪 fd
t1   handle(42): read → 200 字节，追加到 conn[42].rbuf
t2   handle(42): read → -1/EAGAIN        ← 不睡，立刻回来
t3   parse(conn[42]): 凑不出整帧 → 保留 rbuf 和解析状态，return
t4   handle(43) ... handle(169)：剩下 127 个 fd 正常推进
t5   下一轮 epoll_wait
     对端补发字节 → fd 42 再次就绪 → 从 t3 的状态接着解析
```

差别不在吞吐，在**故障隔离**。非阻塞把「一个连接的进度」和「事件循环的进度」解耦：慢连接只是自己慢，不再把别人的延迟一起买单。这也是为什么单线程 Redis、Nginx 的 worker 敢在一条线程上扛住十万连接——它们的每一次 socket 调用都有界。

### 一张「看起来像 / 实际是」对照表

| 流行说法 | 实际语义 |
| --- | --- |
| epoll 说可读，`read` 就不会阻塞 | 只保证入 ready list 的那一瞬有事可做；谎报、被抢、第二次读都能让它睡 |
| LT 模式不需要非阻塞 | LT 免掉的是「必须读干净」的义务，不是「调用不会睡」的保证 |
| 可写事件表示能把我的包写完 | 只表示缓冲区有空间；`large write may still block`，短写是常态 |
| 短写（`write` 返回 `n < len`）是异常 | 是正常返回值，必须自己记 offset 续写 |
| 非阻塞 IO 就是异步 IO | 非阻塞只是「不睡」，数据拷贝仍在你的调用里同步完成 |

## 三、LT 与 ET：一个是义务不同，不是安全性不同

| | LT（水平触发，默认） | ET（边缘触发，`EPOLLET`） |
| --- | --- | --- |
| 触发条件 | 只要缓冲区还有数据就一直报 | 状态发生**变化**时报一次 |
| 没读干净的后果 | 下一轮 `epoll_wait` 继续报，不丢事件 | 可能再也不报，连接静默挂死 |
| 每条消息的系统调用数 | 多（每轮都返回，可能空转） | 少（读到 `EAGAIN` 为止，一次批干完） |
| 非阻塞是否必需 | 强烈建议 | **强制** |
| 写事件管理 | 不需要发数据时要 `MOD` 掉 `EPOLLOUT`，否则忙轮询 | 可以一次注册 `EPOLLIN\|EPOLLOUT` 长期挂着 |
| 多线程唤醒 | 可能惊群（`EPOLLEXCLUSIVE` 缓解） | 只唤醒一个线程 |

ET 强制非阻塞的理由，`epoll(7)` 写得很直白：

> An application that employs the `EPOLLET` flag should use nonblocking file descriptors to avoid having a blocking read or write starve a task that is handling multiple file descriptors. The suggested way to use epoll as an edge-triggered interface is as follows: (1) with nonblocking file descriptors; and (2) by waiting for an event only after `read(2)` or `write(2)` return `EAGAIN`.

逻辑是闭环的：ET 要求你读到 `EAGAIN`，否则丢事件；而「读到 `EAGAIN`」这个终止条件只有非阻塞 fd 才存在。阻塞 fd 上没有 `EAGAIN`，只有 `schedule()`。**ET + 阻塞 fd 不是性能差，是必然死锁**：你要么没读干净等一个不会来的事件，要么在最后那次「确认读干净」的 `read` 上睡过去。

LT 加阻塞则是另一种问题——它不会必然死，只会偶发死，因此更难查。上面四条路径里，LT 只挡掉了「没读干净会丢事件」这一条；谎报、被抢、写不完、以及「读第二次凑整帧」全都还在。更隐蔽的是第三类：即使你老老实实只 `read` 一次，业务解析拿不到完整帧时，如果代码写成「在这儿等齐了再走」，阻塞就从内核挪进了你的解析器。**事件循环被停住这件事，内核睡和业务阻塞是同一个后果。**

### 顺带划清：非阻塞不是异步

这两个词经常被当同义词用，但它们在「谁做数据拷贝」上完全不同：

| 模型 | 谁等 | 数据拷贝发生在 | 典型接口 |
| --- | --- | --- | --- |
| 阻塞 IO | 你的线程睡在内核 | 你的系统调用里 | `read` on blocking fd |
| 非阻塞 + 多路复用 | 你的线程睡在 `epoll_wait` | 你的 `read` 里，同步完成 | `epoll` + `O_NONBLOCK` |
| 信号驱动 | 内核发 `SIGIO` | 你的 `read` 里 | `F_SETOWN` + `O_ASYNC` |
| 真异步 | 不等，提交后拿完成通知 | **内核替你拷**，完成后通知 | `io_submit`（POSIX AIO 长期只对 O_DIRECT 有用）、`io_uring` |

所以 `epoll` 这套东西在 POSIX 的分类里叫 **synchronous I/O multiplexing**（`select(2)` man 页 NAME 段的原话）。你仍然是同步拷数据的那个人，只是从来不睡在拷贝之外的地方。`io_uring` 才是把「等」和「拷」一起交给内核，代价是完全不同的编程模型和内存生命周期管理。

Go 把这两件事都藏在了 runtime 里：`net` 包创建的每个 fd 一定是非阻塞的，并且注册进 netpoller（Linux 上用的是 `epoll` 的 ET 模式）。你写的 `conn.Read(buf)` 看起来像阻塞调用，实际是「非阻塞 read → 拿到 `EAGAIN` → `gopark` 当前 goroutine → 事件到了由 netpoller 唤醒」。阻塞的是 goroutine，不是线程——这正是 [Go 调度器：G、M、P 与 goroutine 调度](/blog/2026/09/02/golang-gmp/) 里那个 P 不被钉死的前提。

## 四、代码：先看怎么死，再看怎么写

### 反面示例：epoll + 阻塞 fd

```c
// 能跑通 demo，能在生产环境挂掉。问题全在注释标出的两行。
int cfd = accept(lfd, NULL, NULL);      // (1) accept 返回的 fd 在 Linux 上不继承
                                        //     listen fd 的 O_NONBLOCK —— 它是阻塞的
struct epoll_event ev = { .events = EPOLLIN, .data.fd = cfd };   // LT
epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);

// ... 事件循环里
for (int i = 0; i < nfds; i++) {
    int fd = events[i].data.fd;
    uint32_t len;
    // (2) 为了凑齐 4 字节长度头而循环 —— 第二次进内核就没数据了，线程睡在这儿
    for (size_t got = 0; got < sizeof(len); ) {
        ssize_t n = read(fd, (char *)&len + got, sizeof(len) - got);
        if (n <= 0) goto close_conn;
        got += n;
    }
    // events[i+1 .. nfds-1] 以及所有新连接，一起等这个对端发第 4 个字节
}
```

`accept(2)` VERSIONS 段专门警告过 (1)：

> On Linux, the new socket returned by `accept()` does not inherit file status flags such as `O_NONBLOCK` and `O_ASYNC` from the listening socket.

把 listen fd 设成非阻塞、却忘了对每个 `accept` 出来的 fd 再设一次，是这个 bug 最常见的来源。它在本机压测里几乎不复现——本机 RTT 低到 `read` 永远都能立刻拿到数据。

### 正面示例：ET + 读到 EAGAIN 的状态机

```c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

// 连接状态：读缓冲 + 写缓冲的进度都得存在这里，
// 因为一次事件处理不保证能做完一件事。
struct conn {
    int  fd;
    char rbuf[64 * 1024]; size_t rlen;            // 已收未解析
    char wbuf[64 * 1024]; size_t woff, wlen;      // 待发送，woff 是已发出的偏移
    int  want_write;                              // 是否已注册 EPOLLOUT
};

static int set_nonblock(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);            // 先读再或；直接 F_SETFL O_NONBLOCK
    if (flags < 0) return -1;                     // 会抹掉 O_APPEND 等其他标志
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

// 返回 0：本轮读干净了（见到 EAGAIN）；-1：对端关闭或出错，调用方负责关 fd
static int drain_read(struct conn *c) {
    for (;;) {
        if (c->rlen == sizeof(c->rbuf)) return 0; // 缓冲满：别再读，否则解析跟不上就是内存炸
        ssize_t n = read(c->fd, c->rbuf + c->rlen, sizeof(c->rbuf) - c->rlen);
        if (n > 0) {
            c->rlen += (size_t)n;
            parse_frames(c);                      // 只解析完整帧，剩下的半包留在 rbuf
            continue;                             // ET 下必须接着读，直到 EAGAIN
        }
        if (n == 0) return -1;                    // 对端 FIN
        if (errno == EINTR) continue;             // 信号打断，重试，不算错误
        if (errno == EAGAIN || errno == EWOULDBLOCK) return 0;
        return -1;
    }
}

// 返回 1：全部写完；0：还有剩余，需要 EPOLLOUT；-1：出错
static int flush_write(struct conn *c) {
    while (c->wlen > 0) {
        ssize_t n = write(c->fd, c->wbuf + c->woff, c->wlen);
        if (n > 0) { c->woff += (size_t)n; c->wlen -= (size_t)n; continue; }  // 短写是常态
        if (errno == EINTR) continue;
        if (errno == EAGAIN || errno == EWOULDBLOCK) return 0;
        return -1;                                // EPIPE / ECONNRESET
    }
    c->woff = 0;
    return 1;
}

static void accept_loop(int epfd, int lfd) {
    for (;;) {
        // accept4 一步拿到非阻塞 + CLOEXEC，省掉一次 fcntl，也省掉忘记设的机会
        int cfd = accept4(lfd, NULL, NULL, SOCK_NONBLOCK | SOCK_CLOEXEC);
        if (cfd >= 0) { register_conn(epfd, cfd); continue; }
        if (errno == EINTR) continue;
        if (errno == ECONNABORTED) continue;      // 这个连接没了，backlog 里可能还有
        if (errno == EAGAIN || errno == EWOULDBLOCK) break;   // backlog 空了，正常退出
        // EMFILE / ENFILE：fd 耗尽。ET 下这里 break 会永久丢掉 listen 事件，
        // LT 下会变成 100% CPU 忙轮询。真实做法是预留一个「牺牲 fd」关掉再 accept，
        // 或者临时 EPOLL_CTL_DEL 掉 listen fd，定时器重挂。此处省略。
        break;
    }
}
```

省略了 `register_conn`、`parse_frames`、epoll 主循环和错误日志，其余可直接编译。几处刻意暴露的裂缝：

- **`rbuf` 满了就停止读**，这会在 ET 下留下一个事件缺口：你没读到 `EAGAIN`，下一轮不会再被通知。所以 `parse_frames` 消费掉数据之后，必须主动把这个 fd 当成「仍然就绪」重新处理一遍（`epoll(7)` 的 "Possible pitfalls" 推荐的做法就是自己维护一份 ready list），或者干脆给这类 fd 退回 LT。没有第三条捷径。
- **`want_write` 的状态机没写全。** `flush_write` 返回 0 时要 `EPOLL_CTL_MOD` 加上 `EPOLLOUT`，返回 1 时要去掉——LT 下不去掉就是每轮空转。ET 下可以一开始就 `EPOLLIN|EPOLLOUT` 长挂，代价是首次注册后要处理一次多余的可写事件。
- **`parse_frames` 在事件循环线程里执行。** 一个连接的解析只要慢，全体都慢。见下面第四个坑。

### Go：什么时候会失手打到阻塞 fd

`net.Conn` 你拿不到阻塞 fd，除非自己动手。两条典型路线：

```go
// 路线 A：从 SyscallConn 拿裸 fd 直接调系统调用
rc, err := conn.(*net.TCPConn).SyscallConn()
if err != nil {
	return err
}
var n int
var rerr error
err = rc.Read(func(fd uintptr) (done bool) {
	n, rerr = syscall.Read(int(fd), buf)
	// 返回 true = "我做完了"，把 EAGAIN 原样交给你自己处理；
	// 返回 false = 让 runtime 把这个 goroutine 挂到 netpoller 上等就绪再重试。
	return true
})
// 这里的 fd 本来就是非阻塞的，所以 rerr 很可能是 syscall.EAGAIN。
// 很多人在这里加一句 SetNonblock(fd, false) "让它变简单" —— 那才是真正的事故。
```

```go
// 路线 B：自己 socket()，默认阻塞
fd, _ := syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, 0)
// 没有 syscall.SetNonblock(fd, true)
syscall.Read(fd, buf)   // 阻塞在内核里
```

路线 B 的后果值得说清楚，因为它和 C 里那个「卡死一万连接」不完全一样。Go 里这次 `syscall.Read` 会把当前 M（OS 线程）钉在内核里；`sysmon` 发现这条 M 进 syscall 太久（本地队列有活立刻抢，否则约 10ms 后）会夺走它的 P 交给另一条 M，其他 goroutine 继续跑。所以**程序不会全停，但你换来了三笔账**：

1. 线程数上涨。每个这样的阻塞调用占一条 OS 线程，上限 `runtime.SetMaxThreads` 默认 10000，撞上就是 `fatal error: thread exhaustion`。
2. `SetDeadline` 失效。deadline 是 netpoller 的能力，裸阻塞 fd 上没人帮你算超时，只剩 `SO_RCVTIMEO` 这类 socket 选项可用。
3. 排查困难。`GODEBUG=schedtrace=1000` 里表现为线程数持续增长、`syscall` 列常驻非零，profile 却看不到 CPU 热点——因为它压根没在跑。

## 五、四个真实业务坑

### 坑 1：listen fd 设了非阻塞，accept 出来的 fd 忘了设

**现象**：压测正常，上线后偶发「某台机器突然不再接受新连接 / QPS 掉成 0，一段时间后自愈」。

**排查**：`cat /proc/<pid>/task/<tid>/stack` 或 `strace -p <pid>`，看线程是不是停在 `read`/`recvfrom`。`ss -nt` 看那条连接的 `Recv-Q` 是 0（对端真没发）还是非 0（你没读）。前者就是这个坑。

**改法**：用 `accept4(lfd, ..., SOCK_NONBLOCK | SOCK_CLOEXEC)` 一步到位，把「记得设非阻塞」从人的记忆变成 API 的约束。同一个坑的变体：`socketpair`、`pipe`、`dup` 出来的 fd，以及从其他进程通过 `SCM_RIGHTS` 收到的 fd，都要单独设。

### 坑 2：ET 下没读干净

**现象**：连接「僵住」——客户端 `ss -nt` 显示 `Send-Q` 为 0（已经发出去了），服务端 `Recv-Q` 非 0（数据在内核里躺着），但服务端进程 CPU 为 0，没有任何日志。客户端重连立刻恢复。

**排查**：`Recv-Q` 非 0 且进程空闲，基本就是事件丢了。对照三处：读循环有没有走到 `EAGAIN` 才退出；有没有因为「一次只处理一个请求」提前 `break`；应用层缓冲满时有没有留补救路径。

**改法**：读循环的唯一合法出口是 `EAGAIN`（或错误/FIN）。做不到（比如要限制单连接每轮的配额来防饿死）就自己维护 ready list，把这个 fd 在下一轮继续当就绪处理。拿不准的服务，直接用 LT——LT 下多几次 `epoll_wait` 返回的开销，远比偶发静默挂死便宜。

### 坑 3：非阻塞 connect 只看可写就当连上了

`connect` 在非阻塞 fd 上返回 `-1` / `EINPROGRESS`，man 页给的收尾方式是明确的：

> It is possible to `select(2)` or `poll(2)` for completion by selecting the socket for writing. After `select(2)` indicates writability, use `getsockopt(2)` to read the `SO_ERROR` option at level `SOL_SOCKET` to determine whether `connect()` completed successfully (`SO_ERROR` is zero) or unsuccessfully.

**现象**：连一个 `ECONNREFUSED` 的地址，连接池却把它当健康连接放进去，直到第一次 `write` 才报错；或者在有防火墙 DROP 规则的路径上，连接静默挂到 `ETIMEDOUT`（`net.ipv4.tcp_syn_retries` 默认 6，约 127 秒）才失败。

**改法**：可写事件之后必须 `getsockopt(SO_ERROR)`，非 0 就关掉重建。另外两点：`connect` 失败后的 socket 状态是未定义的，`connect(2)` NOTES 要求关掉重建而不是原地重试；重试期间再调 `connect` 会拿到 `EALREADY`，别把它当致命错误。超时不要指望内核，自己挂定时器。

### 坑 4：把业务处理放进 epoll 线程

**现象**：平均延迟很漂亮，P99 抖得没规律，抖动幅度恰好等于「最慢的那个请求的处理时间 × 事件批大小」。加机器不管用，因为瓶颈不是 CPU 总量，是那一条线程的串行度。

**排查**：在 `epoll_wait` 返回和下一次 `epoll_wait` 调用之间打一个时长直方图。健康的事件循环这个值应该是几十微秒量级；出现毫秒级尾部，就说明有东西在循环里做了它不该做的事——JSON 反序列化、同步 DB 查询、一次 `log.Println` 打到慢盘、一把被别的线程持有的 `mutex`。

**改法**：事件循环里只做三件事——`read` 到 `EAGAIN`、切帧、把完整帧投进队列；写侧只做 `flush_write` 和 offset 维护。业务处理交给固定大小的 worker 池，队列必须有界并且定义好满了怎么办（见 [Go 任务池：并发限制、排队与退出](/blog/2026/09/14/golang-worker-pool/)）。这条边界画清楚之后，「非阻塞」才是真的——否则你只是把内核的睡眠搬到了用户态。

## 总结

多路复用回答「等谁」，非阻塞回答「这一次调用会不会睡」，前者不蕴含后者：内核可能谎报就绪、数据可能被别的线程拿走、你为了凑整帧的第二次 `read` 也可能扑空。ET 模式把非阻塞从建议变成硬约束（终止条件就是 `EAGAIN`），LT 只免掉「必须读干净」的义务，不免掉阻塞风险。落到代码上只有一条纪律：**事件循环里的每一次系统调用都必须有界返回，每一个连接的进度都必须能存下来下次接着做**——`O_NONBLOCK`、读到 `EAGAIN` 为止、以及给短写留 offset，是这条纪律的三个实现细节。
