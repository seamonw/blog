---
title: C++ 后台面试：对象生命周期、性能剖析、事件模型与分布式失败语义
date: 2026-11-03 16:10:00
categories: Cpp
tags:
	- cpp
	- 面试
	- 并发
	- epoll
	- 分布式
	- 性能
---

十年以上的 C++ 后台面试，拉开差距的通常不是「虚函数表是什么」「epoll 比 select 快在哪」，而是：这段代码在哪一行产生未定义行为，这个接口返回超时之后对方到底做没做，这次优化看的是哪一层的指标。答错的人通常概念都对，只是把两个相邻的保证当成了一个。

本文按四块能力要求，每块只展开决定对错的一两个判据。内存序、分配器、TCP 状态机、epoll 非阻塞、多数派与分布式锁都已有专文，这里只链回去。

| 能力要求 | 面试真正在考 | 容易讲反的一句 |
| --- | --- | --- |
| STL、智能指针、对象生命周期、模板 | 资源什么时候失效，失效后谁还持有它 | 迭代器失效和引用失效是两套规则 |
| Linux 调试与性能剖析 | 一个现象对应哪一层指标 | CPU 高、P99 高、RSS 高是三条不同的调查路径 |
| epoll、Reactor/Proactor、锁与无锁 | 哪条等待队列被唤醒，原子操作发布了哪些写 | 共用一个 epoll fd，水平触发下照样惊群 |
| RPC、服务治理、一致性、高可用 | 返回失败之后效果可能已经生效 | 超时不是「没执行」，是第三态 |

## 一、对象生命周期：迭代器、引用、对象各有各的失效规则

`std::vector` 的判据是**有没有重新分配**：新的 `size()` 超过旧的 `capacity()`（或 `reserve` 了更大的容量）时，所有迭代器、指针、引用失效；没有重新分配时，只有插入点及之后的失效。

```text
push_back(d) 时 size 超过 capacity：

old buffer (freed)             new buffer
+---+---+---+                  +---+---+---+---+---+---+
| a | b | c |      move        | a | b | c | d |   |   |
+---+---+---+  ------------>   +---+---+---+---+---+---+
  ^                              ^
  p (dangling)                   &v[0]
```

扩容前保存的 `p` 指着已释放的旧 buffer。用 ASan 跑，报告是 `heap-use-after-free`，`freed by` 栈里是 libstdc++ 的扩容函数（GCC 15 上叫 `_M_realloc_append`），一眼能和「野指针」区分开。

图里的箭头写的是 move，实际走的是 `std::move_if_noexcept`：元素的移动构造没标 `noexcept`、又能复制时，扩容逐个**复制**，为的是保住 `push_back` 的强异常保证（中途抛异常，旧 buffer 原样不动）；只能移动不能复制时，照样用可能抛异常的移动，强保证随之放弃。另一个更隐蔽的来源：自己写了析构函数的类不会隐式生成移动构造，扩容同样退化成复制，而且包含它的外层类型也一起退化。这些都不报错，只在剖析里表现为扩容变慢。要钉住，就写 `static_assert(std::is_nothrow_move_constructible_v<T>)`。

vector 的规则不能套到别的容器上，而且迭代器失效和引用失效要分开算：

| 容器 | 插入后 | 删除后 |
| --- | --- | --- |
| `vector` | 重新分配：全部失效；否则插入点及之后失效 | 被删位置及之后失效 |
| `deque` | 两端插入：**迭代器全部失效，引用和指针仍有效**；中间插入：全部失效 | 删首或删尾：只失效被删元素（删尾还失效 `end()`）；中间删除：全部失效 |
| `unordered_map` | 触发 rehash 时**迭代器失效，引用和指针仍有效** | 只失效被删元素 |
| `map` / `set` / `list` | 都不失效 | 只失效被删元素 |

`deque` 两端插入和 `unordered_map` rehash 是最常被讲反的两行。循环里删除一律用返回值：`it = c.erase(it)`。

智能指针的对应问题是：对象没了，观察者还在不在。从裸指针再构造一个 `shared_ptr` 会得到**第二个控制块**，两边各析构一次。要拿自己的 `shared_ptr` 必须继承 `enable_shared_from_this`，而且调用时对象已经被某个 `shared_ptr` 接管；栈上对象、构造函数里调用 `shared_from_this()`，都抛 `std::bad_weak_ptr`（C++17 起；之前是未定义行为）。

```cpp
struct Session : std::enable_shared_from_this<Session> {
    // 构造函数里不能调 shared_from_this：此时还没有 shared_ptr 接管 this。
    // 所以异步启动放在 make_shared 之后单独调用的 start() 里。
    void start() {
        auto self = shared_from_this();
        async_read([self](int n) {          // 捕获 self：异步操作完成前对象不会析构
            if (n > 0) self->on_data(n);
        });
    }
    void on_data(int n);
    // async_read 是示意用的异步接口，声明从略
};
```

回调里只捕获 `this`，与「用了智能指针」无关：没人延长对象寿命，回调触发时对象可能已经没了。`shared_ptr` 的线程安全只覆盖控制块计数：不同线程拷贝**不同**的 `shared_ptr` 副本是安全的；一个线程给**同一个** `shared_ptr` 变量赋值、另一个线程同时拷贝它，是数据竞争，要用 C++20 的 `std::atomic<std::shared_ptr<T>>`。对象自己的成员更不受保护，要另外同步。

循环引用用 `weak_ptr` 打断。`lock()` 返回空就是对象已经没了，不要当成暂时错误去重试。

模板那一问，判据是**错误是否发生在替换的直接上下文（immediate context）里**：

| 失败位置 | 结果 |
| --- | --- |
| 声明的直接上下文：返回类型、模板参数、`requires` 子句里的表达式和类型本身 | 候选被剔除，重载决议继续 |
| 声明触发的其他模板实例化内部：返回类型写 `typename Wrap<T>::type`，`Wrap<T>` 里的 `static_assert` 失败 | 硬错误，写在返回类型里也一样 |
| 函数体里 | 候选已经选中，实例化时硬错误 |

```cpp
template <class T> auto g(T t) -> decltype(t.size(), int()) { return 1; }  // 声明里失败：剔除
int g(...) { return 0; }

template <class T> int h(T t) { return t.size(); }                          // 函数体里失败
int h(...) { return 0; }

int a = g(1);  // 0：模板版本被剔除，落到可变参数版本
int b = h(1);  // 编译错误：模板 h 比 h(...) 更匹配，选中之后才实例化函数体
```

concept 没有改变这条规则，只是把剔除条件写成有名字的约束，报错时指向没满足的那一条。另一个随编译器版本变化的细节：`if constexpr` 的丢弃分支里写 `static_assert(false)`，GCC 12 直接报错，GCC 13 起接受（P2593 / CWG2518，作为缺陷报告回溯，`-std=c++17` 下同样通过）。要兼容旧编译器，写成依赖模板参数的假条件，例如 `always_false<T>::value`。

## 二、Linux 调试与性能：先定是哪一层在痛，再选工具

同一个「接口变慢」，三种读数对应三条路径。混在一起看，后面的工具都是在证明一个已经错的假设。

| 现象 | 先看 | 再看 | 不要先做的事 |
| --- | --- | --- | --- |
| 单核打满，吞吐量上不去 | `perf top` / `perf record -g` 的热函数 | 热点是业务、分配器还是自旋锁 | 直接改算法，热点可能是 `malloc` |
| CPU 不高，P99 很高 | off-CPU：线程睡在哪 | 锁（`futex`）、磁盘、等对端，还是 cgroup CPU 限流 | 用 on-CPU 采样，等待时间根本不进样本 |
| RSS 持续涨，业务量平稳 | 堆剖析，按分配栈聚合 | 泄漏还是分配器缓存不还页 | 把缓存当泄漏修 |

第三行最容易讲反。用户态分配器的 `free` 把块放回缓存，不一定把页还给内核，RSS 不降是设计如此。判据是：泄漏的分配栈随时间累积、没有上界；缓存会稳定在某个档位。ptmalloc 上有个便宜的判别法：调一次 `malloc_trim(0)`（glibc 2.8 起会遍历所有 arena，把整页空闲的块还回去），RSS 明显回落就是缓存。三种分配器各自什么时候还页，见 [三种用户态分配器：ptmalloc、tcmalloc 与 jemalloc 的缓存和还页](/blog/2026/10/30/three-allocators/)。

第二行的锁争用，不要指望 `perf lock`，它主要分析内核锁事件。`std::mutex` 争用到最后是 `futex` 等待：off-CPU 火焰图里用户态那一截是 `__lll_lock_wait` / `pthread_mutex_lock`，往下几帧就是哪把锁。自旋锁相反，把等待烧成 CPU，会伪装成第一行。

第二行在容器里还有一个常被漏掉的来源：CPU 配额限流。cgroup 的 `cpu.max`（v1 是 `cpu.cfs_quota_us` / `cpu.cfs_period_us`）按周期发配额，默认周期 100ms。多线程服务在周期前段就把配额烧完，剩下的时间所有线程一起被挂起，P99 被抬高几十毫秒，平均 CPU 利用率却离上限很远。它在 off-CPU 火焰图里不集中在 `futex` 上，而是散在业务代码各处的被抢占点，容易被当成普通的调度排队放过。判据看容器的 `cpu.stat`：`nr_throttled` 和 `throttled_usec`（v1 是 `throttled_time`）跟着延迟毛刺一起涨。

工具按「回答什么问题、代价多大」选：

| 工具 | 回答的问题 | 代价 | 线上 |
| --- | --- | --- | --- |
| `perf record -g` | CPU 花在哪 | 采样，开销随频率可控 | 可以 |
| off-CPU（bcc `offcputime`、`perf sched`） | 线程在等什么 | 跟踪调度事件，切换频繁时开销明显 | 短时间开 |
| ASan | 越界、UAF、double free | 需重新编译，clang 文档给的典型减速约 2 倍 | 测试、压测 |
| TSan | 数据竞争 | 典型减速 5–15 倍，内存 5–10 倍 | 否 |
| Valgrind memcheck | 越界、泄漏、**未初始化值的使用**（ASan 不报这类），无需重编 | 官方 Quick Start：慢 20–30 倍 | 否 |
| GDB / `gcore` | 某个时刻每个线程停在哪 | attach 或转储期间整个进程停住 | 慎用 |

线上看堆用采样型剖析（tcmalloc heap profile、jemalloc `prof`），不要挂 memcheck。

GDB 看的是一个时刻。进程卡死，先 `thread apply all bt` 看是不是都堆在同一把锁上。libstdc++ 的 `std::mutex` 包着 `pthread_mutex_t`（成员 `_M_mutex`），`p mtx._M_mutex.__data.__owner` 就是持锁线程的 LWP 号，一步找到持锁者再看它的栈。线上 `gdb -p` 期间整个进程停住，健康检查超时会被摘流量甚至被拉起；用 `gcore` 留一份 core 离线分析，停顿只有写 core 那一段。

## 三、高并发：先问哪条等待队列被唤醒，再问原子操作发布了什么

### 惊群：两个来源，两种解法

惊群要按「哪条等待队列」分开说。N 个 epoll 实例都加了同一个 listen fd 时，listen socket 的等待队列上挂着 N 个回调，一条新连接把它们全部唤醒：

```text
一个 listen fd 被加进 N 个 epoll 实例（每线程一个 epoll 的常见写法）

                 new connection
                       |
        +--------------+--------------+
        v              v              v
     epoll A        epoll B        epoll C      全部被唤醒
        |              |              |
     accept()       accept()       accept()
       ok            EAGAIN         EAGAIN
```

另一个来源常被说反：N 个线程在**同一个** epoll fd 上 `epoll_wait`，水平触发（默认）下同样会接连唤醒多个线程。就绪项交给第一个线程后又被放回就绪链，内核接着唤醒下一个等待者。epoll(7) 只承诺 `EPOLLET` 下「just one of the threads」被唤醒。

| 写法 | 一条新连接唤醒几个 | 代价 / 限制 |
| --- | --- | --- |
| N 个 epoll 实例 + 同一 listen fd | 全部 | 白醒 N−1 个 |
| 同上，`ADD` 时带 `EPOLLEXCLUSIVE`（4.5+） | 一个或几个（man 原话是 one or more） | 只能在 `ADD` 时带；accept 队列仍是一条 |
| N 线程共用一个 epoll fd，水平触发 | 可能接连醒多个 | 就绪项放回后接力唤醒 |
| 同上，`EPOLLET` | 一个 | 醒来的线程要 `accept` 到 `EAGAIN`，负载不均摊 |
| `SO_REUSEPORT`（3.9+），每线程一个 listen fd | 只有分到连接的那个 | 按哈希分配，不看负载 |

`EPOLLEXCLUSIVE` 的两个限制要能说出来，否则等于没用过：之后对同一对 `(epfd, fd)` 做 `EPOLL_CTL_MOD` 返回 `EINVAL`；它只能和 `EPOLLIN`、`EPOLLOUT`、`EPOLLET`、`EPOLLWAKEUP` 组合，再加 `EPOLLONESHOT` 同样是 `EINVAL`。所以它解决不了「两个线程同时处理同一个已建立连接」。那件事归 `EPOLLONESHOT`：事件投递一次后该 fd 在兴趣列表里被禁用，处理完必须 `EPOLL_CTL_MOD` 重新装上。

`SO_REUSEPORT` 有两条裂缝。第一，内核按四元组哈希挑 listener，不看谁忙：一个线程卡在长请求上，分给它的连接照样进它的队列排队；队列满后新 SYN 被丢（`netstat -s` 里 listen queue overflow 计数上涨），客户端 1 秒后才重传 SYN。第二，关掉其中一个 listener（滚动重启、缩线程），它队列里握手中和已完成握手的连接都被 abort。Linux 5.14 加了 `net.ipv4.tcp_migrate_req`，打开后才会迁到同端口的其他 listener，默认关闭。这几种组合逐一走过的长文见 Marek Majkowski 的 [Epoll is fundamentally broken](https://idea.popcount.org/2017-02-20-epoll-is-fundamentally-broken-12/)。

已建立连接上的读写仍然要非阻塞，边沿触发必须读到 `EAGAIN`，见 [为什么 IO 多路复用必须搭配非阻塞 IO](/blog/2026/09/16/io-multiplexing-nonblocking/)。握手队列、重传与拥塞见 [TCP/IP 面试全解：握手挥手状态机、丢包重传、滑动窗口与拥塞控制](/blog/2026/10/30/tcp-interview/)。

### Proactor：取消不等于结束

Reactor 通知「可以读了」，buffer 在你调 `read` 时才出场；Proactor（IOCP、io_uring）提交时就把 buffer 交给内核，完成之前它不属于你。概念对照见 [操作系统面试全解：虚拟内存、进程线程、Page Cache 与零拷贝](/blog/2026/10/30/os-interview/)。这里只讲决定对错的一点：生命周期问题从 fd 挪到了 buffer，而取消是有竞态的。

```text
submit read(fd, buf)            此后 buf 归内核
      |
  超时，submit cancel(user_data)
      |
      +--> cancel 的 CQE：  0 | -ENOENT | -EALREADY
      +--> 原请求的 CQE：  -ECANCELED | 正常结果（数据已写进 buf）
```

按 io_uring_prep_cancel(3)，cancel 的三种结果分别是：

- `0`：原请求已被取消，它的 CQE 在 cancel 提交返回前已经投递，但你可能还没收割到。
- `-ENOENT`：没找到，通常是原请求在 cancel 之前已经完成，它的 CQE 同样可能还在完成队列里。
- `-EALREADY`：已经走到不能取消的阶段，原请求稍后才完成，结果可能是成功；这段时间内核仍可能在写 buffer，man 原文要求调用方等原请求自己的 CQE。

只有一条规则在所有情况下都安全：**buffer 和持有它的上下文跟着原请求的 CQE 释放，不跟 cancel 的结果走**。按「cancel 回来就 free」写，就是 io_uring 版的 use-after-free。

Boost.Asio 的接口是 Proactor 风格（`async_read` 带完成回调），但 Linux 上的默认实现仍是 epoll Reactor。io_uring 后端从 Boost 1.78 起才有，默认关闭，要同时定义 `BOOST_ASIO_HAS_IO_URING` 和 `BOOST_ASIO_DISABLE_EPOLL` 才接管全部 I/O。说「用了 Asio 就是 Proactor」，只对了接口那一层。

### 线程池与锁：先治队列和临界区

Reactor 线程里出现一次阻塞调用（同步 DNS、日志刷盘、等一把可能被长时间持有的锁），这个线程上所有连接的延迟一起抬高，CPU 却不高，正好是第二节表格的第二行。阻塞的活交给线程池，结果通过 `eventfd` 或任务队列送回 Reactor 线程，连接状态只在一个线程上改。

线程池的判据是队列。无界队列把过载变成延迟和内存，而不是错误；队列有界、满了怎么处理（拒绝、调用方自己执行、丢最旧），就是这个服务的过载策略。线程数按「核数 ×（1 + 等待时间 / 计算时间）」估只是起点：等待时间会随下游变慢而变，最终要靠在途数上限兜住（见第四节）。Go 版的并发上限、排队与退出见 [Go 任务池：并发限制、排队与退出](/blog/2026/09/14/golang-worker-pool/)。

锁优化的顺序：临界区里不做 I/O、不分配内存、不调用户回调；再按 key 分片；最后才考虑无锁。读写锁不是读多写少的默认答案：读临界区很短时，每个读者都要原子地改同一个读者计数，这条 cache line 在核之间来回弹，争用本身就是全部开销，常常比普通 mutex 还慢。同理，两个被不同线程频繁写的变量落在同一条 cache line 上（伪共享），没有任何锁也会互相拖慢。

填充按多少字节，没有普适常量。x86 的 cache line 是 64 字节，但 Intel 的空间预取器会把 128 字节对齐的相邻两条一起拉取，所以 folly、crossbeam 在 x86-64 上按 128 字节填充；Apple M 系列的 cache line 本身就是 128 字节。标准里对应的是 C++17 的 `std::hardware_destructive_interference_size`（GCC 12 起提供），但它是编译期常量，一旦进了头文件或 ABI，就固定成编译那台机器的取值，GCC 为此专门加了 `-Winterference-size` 警告。

### 无锁：原子操作发布了哪些写

`compare_exchange` 或 `exchange` 成功，只说明这一个原子对象换成了新值。它不保证你在这之前写的普通内存也被别的线程看见，除非这次写带 release（或更强）、对方用 acquire 读到它。各种 `memory_order` 的差别见 [C++ memory order：多线程下的可见性与同步](/blog/2026/08/28/cpp-memory-order/)。

下面这个 MPSC 队列只靠一对 release / acquire 成立：生产者先写好 `value`，再用 release 把节点挂上链；消费者用 acquire 读到这个挂接，才能读 `value`。

```cpp
#include <atomic>

// 多生产者、单消费者（Vyukov 式侵入链表）。只有一条线程调 pop。
// 示意代码：没写析构，退出时链上剩余节点和当前 stub 由调用方回收。
struct Node {
    std::atomic<Node*> next{nullptr};
    int value = 0;
};

class Mpsc {
public:
    explicit Mpsc(Node* stub) : head_(stub), tail_(stub) {}

    void push(Node* n) {                                   // 任意线程，调用前已写好 n->value
        n->next.store(nullptr, std::memory_order_relaxed);
        // acq_rel：上一个生产者对 prev->next 的初始化，先于本线程下面那次写
        Node* prev = head_.exchange(n, std::memory_order_acq_rel);
        // 窗口：n 已经是新头，但还没挂到链上
        prev->next.store(n, std::memory_order_release);   // 发布 n->value
    }

    Node* pop() {                                          // 只能消费线程调
        Node* t = tail_;
        Node* n = t->next.load(std::memory_order_acquire); // 与上面的 release 配对
        if (n == nullptr) return nullptr;                  // 真空，或某个生产者停在窗口里
        tail_ = n;                                         // n 成为新的 stub
        t->value = n->value;                               // 值搬到旧 stub 上交给调用方
        return t;
    }

private:
    alignas(64) std::atomic<Node*> head_;  // 生产者争抢
    alignas(64) Node* tail_;               // 消费者独占；分开 cache line，避免伪共享（64 按 x86 取，见上一节）
};
```

用 GCC 15.2 `-fsanitize=thread` 跑 4 个生产者、各 5 万次入队：上面的版本无报告，20 万个值的总和校验通过；只把发布那一行改成 `relaxed`，TSan 报 `pop` 读 `n->value` 与生产者线程初始化该节点之间的数据竞争。这个 bug 在 x86 上多半照常跑过：硬件不重排 store 与 store，只剩编译器重排一种出错方式。TSan 按 C++ 内存模型判，不按硬件判。

这段代码有三条裂缝，面试里主动说出来比写对代码更加分：

- **`pop` 返回的是旧 stub**，不是你 push 进去的那个指针。节点的归还和释放要按这个约定来，第一次 `pop` 拿到的是构造时传入的 stub。
- **空有两种含义**：真空，或者某个生产者停在 `exchange` 和 `store` 之间。后者更麻烦：在它之后入队的节点都挂在它后面，消费者一个也看不到，直到它被调度回来。所以入队只有一次 `exchange`、从不循环重试，是 wait-free；出队却会被一个被抢占的生产者挡住，按进展保证的定义是阻塞的，连 lock-free 都算不上。没有 mutex 不等于 lock-free：lock-free 要求任意线程停在任意一步时，系统整体仍有进展。
- **单消费者是它没有 ABA 的原因**。换成多消费者，`pop` 要对 `tail_` 做 CAS，难点立刻变成节点什么时候能释放：另一个消费者可能刚读到这个指针，还没解引用。答案是 hazard pointer、epoch / RCU 这类延迟回收。C++26 已把 `<hazard_pointer>`（P2530R3）和 `<rcu>`（P2545R4）收进标准库，但截至本文写作（2026 年 11 月），libstdc++ 和 libc++ 都还没有这两个头文件，生产上用 folly 的 `hazptr` / `rcu`。别拿 `std::atomic<std::shared_ptr>` 充当回收方案：libstdc++ 的实现带锁，GCC 15.2 上 `is_lock_free()` 返回 false。

## 四、分布式：超时之后对方可能已经做完了

一次 RPC 有三种结局，不是两种。响应里的成功或业务错误是确定的；超时、连接断开、进程崩溃都只说明**响应没有到达调用方**，请求可能没发出去，也可能已经执行完并提交。把超时当失败然后直接重试，会把一次扣款执行两次。

```text
调用方                          服务方
  |  请求（带幂等键）
  | ----------------------------> 执行并提交
  |                               响应在这里丢失
  |  超时。此时不知道上面发生了没有
  |
  |  用同一个幂等键再请求一次
  | ----------------------------> 命中已提交的记录，返回当时的结果
  | <---------------------------- 不再执行一次
```

幂等的前提是服务方把「幂等键 → 结果」和业务修改写进**同一次提交**。两者分开提交，崩溃窗口里业务已经生效、记录还没有，重试仍然会执行第二次。只读接口不需要这一套；不可重复的写必须有。

重试放大是乘法。调用链三层、每层最多试 3 次，最底层最多收到 3 × 3 × 3 = 27 次请求，而它此刻多半正是过载的那一个。Google SRE 书 *Handling Overload* 一章给的上界：单个请求最多 3 次尝试；按客户端统计重试占比，超过 10% 就不再重试。工程上再加一条：只在一层重试，下层过载时返回明确的「过载，勿重试」错误。超时也一样，要从调用方的截止时间往下传（gRPC 用 `grpc-timeout` 头携带剩余时间）；每一层各自计时，就会出现上游已经放弃、下游还在执行的请求。

服务治理那一问，判据是**限流按在途数比按 QPS 稳**。在途请求数 = QPS × 延迟（Little 定律）：下游变慢时，同样的 QPS 意味着更多在途请求，占住更多线程和连接。按 QPS 限流的服务这时仍然放行，直到线程池耗尽；按在途数限流会随延迟上升自动收紧。熔断解决的是另一件事：调用方在下游错误率高时短路，保护的是自己的线程和下游的恢复时间，不能替代限流。

一致性那一问是同一个结构：多数派只约束已经完成的写。写还停在少数节点上时，读可以先看到它，下一次读又可能看不到，见 [线性一致性：多数派读写为什么还会出现新值变旧值](/blog/2026/10/28/linearizability-quorum/)。提交点、租约到期与旧持有者的写，见 [分布式事务与分布式锁：提交记录和 fencing token](/blog/2026/10/29/distributed-tx-lock/)。

## 避坑

- **迭代器 / 指针失效当成偶发崩溃**。循环 `push_back` 之前保存了 `&v[0]` 或迭代器，某次扩容后悬空，崩溃位置随机。排查：ASan 报 `heap-use-after-free`，`freed by` 栈里是 vector 的扩容函数（libstdc++ 新版本叫 `_M_realloc_append`，旧版本叫 `_M_realloc_insert`）。改法：先 `reserve` 到上界再取地址，或者存下标。
- **`SO_REUSEPORT` 滚动重启时掉连接**。每次发布都有一小撮客户端报 connection reset，时间点和旧进程关闭 listener 对得上。原因是被关 listener 队列里的连接被 abort。改法：内核 ≥ 5.14 打开 `net.ipv4.tcp_migrate_req`；或者在新旧进程之间传递 listen fd（`SCM_RIGHTS`、systemd socket activation），让 listener 从不关闭。
- **超时当失败重试**。日志里同一业务单号有两次成功提交，中间隔着一次客户端超时。改法：写接口带幂等键，键与业务同一事务提交；只在一层重试、有预算，并向下传剩余截止时间。

## 面试重点

| 追问 | 答到这里才算过 |
| --- | --- |
| `deque::push_back` 之后，之前拿到的元素引用还能用吗 | 能。迭代器全部失效，引用和指针不受影响 |
| 构造函数里能调 `shared_from_this()` 吗 | 不能，抛 `std::bad_weak_ptr`：这时还没有 `shared_ptr` 接管它 |
| 返回类型里的替换失败和函数体里的错误有什么区别 | 直接上下文里的替换失败剔除候选、继续重载决议；函数体里出错，或声明触发的其他模板实例化内部出错，都是硬错误 |
| N 个线程共用一个 epoll fd，还会惊群吗 | 水平触发会；`EPOLLET` 才保证只醒一个 |
| `EPOLLEXCLUSIVE` 能配 `EPOLLONESHOT` 吗 | 不能，`EINVAL`；它也只能在 `ADD` 时带，不能 `MOD` |
| 这个 MPSC 队列是 lock-free 吗 | 不是。push 是 wait-free；pop 会被停在 `exchange` 与 `store` 之间的生产者挡住，是阻塞的 |
| io_uring 的 cancel CQE 回来了，buffer 能释放吗 | 不按它判断：`-EALREADY` 时内核还在写；统一等原请求自己的 CQE |
| 超时的写请求能重试吗 | 只有带幂等键、键与业务同一事务提交时才能，且重试有预算 |

### 两道核心题的三层追问

上表每题只到第一层。下面两题最能拉开差距，各往下追两层。

io_uring 的 buffer 什么时候能释放：

| 层 | 追问 | 答到这里才算过 |
| --- | --- | --- |
| 1 | cancel 的 CQE 回来了，buffer 能释放吗 | 不按它判断：`-EALREADY` 时内核还在写；统一等原请求自己的 CQE |
| 2 | 进程要退出，还有一堆在途请求，怎么收尾 | 先发 `IORING_ASYNC_CANCEL_ANY`（5.19+）取消全部，再把所有原请求的 CQE 收割干净，最后才释放 buffer、上下文和 ring；顺序反过来就是退出路径上的 UAF |
| 3 | 换成 multishot recv，一个请求多次完成，规则怎么变 | 一个请求对应多个 CQE，只有不带 `IORING_CQE_F_MORE` 的那个才是最后一个，上下文跟它释放；带 `F_MORE` 的 CQE 只交付数据，不触发释放 |

超时之后能不能重试：

| 层 | 追问 | 答到这里才算过 |
| --- | --- | --- |
| 1 | 超时的写请求能重试吗 | 只有带幂等键、键与业务同一事务提交时才能，且重试有预算 |
| 2 | 幂等记录存多久；同一个键带着不同参数又来了怎么办 | 保留期至少覆盖客户端最长的重试窗口（截止时间加上全部重试和退避），过期后同键请求会被当成新请求执行；记录里存请求摘要，同键不同参数直接报错，不能返回旧结果冒充成功 |
| 3 | 下游是不支持幂等键的第三方通道，怎么办 | 不能盲重试。超时后把订单落成「结果未知」状态，用对方的查询接口按自己的业务单号确认结果，再决定是否重发；对方连查询都没有，就只能靠对账兜底，并在设计评审里把这条风险写明 |

## 总结

迭代器、`shared_ptr`、io_uring 的 buffer 是同一个问题的三种形态：谁还持有它，它什么时候失效。性能和并发问题先定层次：是 CPU、等待还是堆，是哪条等待队列被唤醒，原子操作发布了哪些写。分布式写接口的超时是第三态，只有幂等键与业务同一事务提交、重试有预算，重试才不会把一次故障放大成雪崩。
