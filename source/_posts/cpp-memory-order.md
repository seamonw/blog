---
title: C++ memory order：多线程下的可见性与同步
date: 2026-08-28 19:40:00
categories: Cpp
tags:
	- cpp
	- memory-order
	- 并发
	- atomics
---

单线程里，C++ 给你一张相当友好的假象：源码从上往下执行，前面的赋值后面一定看得见。多线程把这张假象撕掉。一个线程刚写完的变量，另一个线程可能读到旧值，甚至读到「按源码顺序不该同时出现」的组合。

`std::memory_order` 不是六种「快一点 / 慢一点」的旋钮，而是在问一件事：**这次原子操作，要不要、以及以何种方式和别的线程建立同步关系。** 可见性是结果，同步是手段。把这两件事分开，memory order 才好讲。

本文从硬件为什么会乱序说起，再落到 C++ 内存模型里的 happens-before / synchronizes-with，最后逐个看六种 `memory_order`，以及它们在锁、消息传递、引用计数里怎么用。

## 单线程的假象从哪来

下面这段在一个线程里永远成立：

```cpp
int x = 0;
int y = 0;
x = 1;
y = 2;
assert(x == 1);  // 一定过
```

编译器可以重排无关的读写，CPU 也可以把 store 先扔进 store buffer。你仍然看见「先写 x 再写 y」，因为**同一线程对自己的写，必须表现得像按程序顺序发生**。这叫 *as-if* 规则：只要单线程观察不到区别，底下怎么折腾都行。

另一个线程没有这层保护。它看到的是缓存行、写缓冲和失效队列拼出来的世界，不是你的源码顺序。

## 另一核看见的不是「内存」

现代 CPU 每个核有私有 L1/L2，共享的是更远的缓存或内存。一致性协议（常见是 MESI 及其变体）保证：**同一时刻，一个缓存行至多有一个核处于 Modified**。这保证的是「不会永远各写各的」，不保证「你写完的下一个周期别人就能读到」，更不保证「两个地址的写按你源码的顺序到达」。

真正让可见性滞后的，往往是核内部的队列，不是缓存协议本身。

```text
核 A                              核 B
-----                             -----
store x = 1                       load y
  └─ 进 store buffer，还没到缓存    load x
store y = 1
  └─ 也可能还在 store buffer

核 B 可能先看见 y==1、再看见 x==0
```

**Store buffer**：store 先在本地排队，load 可以绕过它去读缓存。对本核，转发逻辑会让你读到自己刚写的值；对别的核，这些 store 还没变成一次缓存一致性事务。

**失效队列 / 读缓冲**：别的核发来的 invalidate 可以先挂着，本核继续用手里那份旧缓存行。

再加上编译器：如果它认为 `x` 和 `y` 不相干，完全可以把 `x=1` 和 `y=1` 对调，甚至把某个写优化没了。`volatile` 能阻止编译器把对**这个对象**的访问优化掉，但：

- 它不提供原子性（`++` 仍可能撕裂）
- 它不建立线程间同步
- 它不能阻止 CPU 的 store buffer 乱序

C++ 里管线程间同步的是 `std::atomic` 和互斥量，不是 `volatile`。`volatile` 留给内存映射寄存器这类「读写本身就有外部副作用」的东西。

## 数据竞争：模型首先禁止的事

C++ 内存模型有一条硬规则：对同一个非原子对象，两个线程一个写、一个读或写，且这两次访问没有 happens-before 关系，这就是 **data race**。数据竞争是未定义行为。程序可以改你的无关变量，可以让 assert 在优化后消失，不存在「大多数时候能跑」这种保证。

所以讨论 memory order 之前，先把对象分成两类：

- **非原子对象**：普通的 `int`、结构体、指针指向的那块堆。它们的跨线程访问必须被同步兜住。
- **原子对象**：`std::atomic<T>`。对它的读写本身不是数据竞争，即使只用 `relaxed`。原子性解决的是「这一次读写会不会撕成两半」，不自动解决「旁边那些普通变量别人看不看得见」。

互斥量、`std::atomic` 的 acquire/release、线程的 join/start，都是在普通数据周围补上 happens-before，让数据竞争不出现。

## 三套顺序，不要混成一个词

C++ 标准用几套不同的「先后」说话。混在一起，memory order 就会变成玄学。

**Sequenced-before（程序顺序）**  
同一线程里，源码规定的先后。`a=1; b=2;` 里，`a=1` sequenced-before `b=2`。这是单线程的顺序。

**Synchronizes-with（同步于）**  
跨线程的那座桥。典型情况：线程 A 对原子对象 M 做了一个 release store，线程 B 对**同一个** M 做了一个 acquire load，并且 B 读到了 A 写入的那个值。这时 A 的那次 store synchronizes-with B 的那次 load。

互斥量也是这座桥：`unlock` synchronizes-with 下一次成功的 `lock`。`thread::join`、`notify` / `wait` 同理。

**Happens-before（先发生于）**  
真正决定可见性的关系。它由程序顺序和 synchronizes-with 拼起来：

```text
同一线程：
  A sequenced-before B  ⇒  A happens-before B

跨线程：
  A synchronizes-with B  ⇒  A happens-before B

传递：
  A happens-before B，B happens-before C  ⇒  A happens-before C
```

结论就一句：

> 如果 A happens-before B，那么 A 对内存做的副作用，在 B 看来都已经完成。B 不能读到 A 之前的旧值。

**可见性**是这个结论的另一面：没有 happens-before，一次写对另一个线程是否可见，模型不保证。你可能碰巧看见，也可能永远看不见，编译器还可以假设你看不见而把代码删掉。

**同步**是在建立 happens-before。memory order 选的就是这座桥要不要建、建多宽。

```text
线程 A                         线程 B
x = 42;                        if (flag.load(acquire)) {
flag.store(true, release);         // 能看见 x == 42
                               }
```

`x=42` 在 A 里 sequenced-before 那个 release store；release store synchronizes-with B 的 acquire load；于是 `x=42` happens-before B 读 `x`。`x` 自己不是原子的，也不需要是——桥搭在 `flag` 上，普通数据跟着过桥。

这就是「可见性靠同步，同步靠原子上的 acquire/release，不靠把所有数据都改成 atomic」。

## 六种 memory_order

`std::atomic` 的 load / store / RMW 都带一个 `memory_order` 参数，默认是 `seq_cst`。从弱到强看。

### relaxed：只要原子，不要桥

```cpp
std::atomic<int> cnt{0};

// 线程们
cnt.fetch_add(1, std::memory_order_relaxed);
```

`relaxed` 保证：

- 这一次读写是原子的，不会读到半写入
- 对**同一个**原子对象，同一个线程里的操作不能被重排到违反修改序（modification order）
- 所有线程对这一个原子对象的 RMW 会排成一条总序

它不保证：

- 这次操作和周围任何普通读写的顺序
- 其他线程何时能看见这次写
- 任何 happens-before

适合「我只关心这个计数本身，不靠它给别的数据发信号」。统计值、近似计数、引用计数的**增加**一侧，常常是 `relaxed`。

反例：用 `relaxed` 当完成标志。

```cpp
int data = 0;
std::atomic<bool> ready{false};

// 线程 A
data = 42;
ready.store(true, std::memory_order_relaxed);  // 没有发布 data

// 线程 B
if (ready.load(std::memory_order_relaxed)) {
    use(data);  // 数据竞争，可能看见 0
}
```

`ready` 变成 `true` 和 `data` 变成 `42`，在 B 看来没有固定顺序。B 甚至可能看见 `ready==true` 而 `data` 仍是 0。

### release / acquire：消息传递

这是时间轮、无锁队列、把普通数据交给另一个线程时最常用的一对。

- **release store**：不能把**程序顺序上在它前面的**读写，挪到这个 store 后面。它「发布」此前所有写入。
- **acquire load**：不能把**程序顺序上在它后面的**读写，挪到这个 load 前面。它「接收」对方发布的那些写入。

两者必须配在**同一个原子对象**上，而且 acquire 一侧必须读到那次 release 写入的值。配错对象、或者根本没读到那个值，桥就不存在。

```cpp
struct Packet { int seq; int payload; };

Packet buf;
std::atomic<bool> published{false};

// 生产者
buf = Packet{1, 99};
published.store(true, std::memory_order_release);

// 消费者
if (published.load(std::memory_order_acquire)) {
    assert(buf.seq == 1);      // 合法
    assert(buf.payload == 99); // 合法
}
```

可以把它想成一个单向阀门：

```text
生产者：  [普通写入……] ──release──▶  flag
消费者：                         flag  ──acquire──▶  [普通读取……]
```

release 不约束它**之后**的操作，acquire 不约束它**之前**的操作。所以下面这样是错的：

```cpp
ready.store(true, std::memory_order_release);
data = 42;  // 这个写在 release 后面，没有被发布
```

以及：

```cpp
use(data);  // 这个读在 acquire 前面，不受保护
if (ready.load(std::memory_order_acquire)) { ... }
```

单向同步还有一个直接推论：两个 `relaxed` 不行，一个 `release` 配一个 `relaxed` 也不行。消费者必须 acquire，否则它后面的读可以被提到 flag 前面去。

### acq_rel：RMW 上的一座双向桥

`memory_order_acq_rel` 只能用在 read-modify-write 上：`exchange`、`compare_exchange_*`、`fetch_add` 这类。对这次 RMW 来说：

- 读的那一半是 acquire
- 写的那一半是 release

典型场景是无锁栈的 `compare_exchange`、或者引用计数的**最后一次减一**。一次 RMW 既要看见前人写进对象里的东西，又要把「我改完了」发布给后人。

单独的 load 不能标 `acq_rel`，单独的 store 也不行。load 用 acquire，store 用 release。

### seq_cst：所有人共用一根时间轴

`seq_cst` 包含 acquire/release 的同步，另外多一件事：程序里**所有** `seq_cst` 操作排成一条全局总序。每个线程看见的 `seq_cst` 操作顺序，都和这条总序一致。

acquire/release 做不到这一点。经典例子是 Dekker 的握手：

```cpp
std::atomic<int> x{0}, y{0};
int r1, r2;

// 线程 A                    // 线程 B
x.store(1, seq_cst);         y.store(1, seq_cst);
r1 = y.load(seq_cst);        r2 = x.load(seq_cst);
```

如果四处都是 `seq_cst`，`r1==0 && r2==0` 不会发生。全局总序里，两个 store 总有一个在前；在后的那个 load 必须看见先发生的那个 store。

把四处都改成 release store + acquire load，`r1==0 && r2==0` 就合法了。两边都可以先把自己的 store 留在 store buffer 里，再读对方还是 0。acquire/release 只同步「我发布的普通数据」和「读到这个 flag 的人」，并不把两个毫不相干的原子变量排进同一条世界线。

另一个常被提起的是 IRIW：四个线程，两个写不同变量，另外两个以相反顺序读。只有 `seq_cst` 禁止「两个读者看到相反的 store 顺序」。

默认参数是 `seq_cst`，不是因为日常都需要这根全局时间轴，而是因为它最容易讲对。代价是：x86 上原子 store 往往要变成 `xchg` 或后面跟 `mfence`，比 release store 贵；ARM / POWER 上差距更明显。

什么时候真的需要它：协议里有「双方各写各的标志、再读对方」这种对称握手，而你又无法改成「谁先发布、谁后观察」的单向模式。其余时候，acquire/release 通常就够。

### consume：数据依赖，实际上已被放弃

`memory_order_consume` 本意是：只要后续读**数据依赖于**这次 load 的返回值（例如 `p->field`，`p` 来自这次 load），就只要这层依赖上的可见性，不必做完整的 acquire 屏障。DEC Alpha 这类会把依赖打散的架构才真正需要把它和 acquire 区分开。

今天的编译器几乎都把 `consume` 当成 `acquire` 实现。标准也在往「弃用 consume、改用 `[[carries_dependency]]` 之外的东西」走。新代码不要写 `consume`。需要依赖序的时候，写 acquire。

### 对照

| order | 用在 | 原子性 | 建立 synchronizes-with | 额外保证 |
| --- | --- | --- | --- | --- |
| `relaxed` | load/store/RMW | 有 | 无 | 单对象修改序 |
| `consume` | load | 有 | 依赖序（形同虚设） | 不要用 |
| `acquire` | load/RMW | 有 | 作为桥的读端 | 后面的访问不能提前 |
| `release` | store/RMW | 有 | 作为桥的写端 | 前面的访问不能推后 |
| `acq_rel` | RMW | 有 | 两端都是 | acquire + release |
| `seq_cst` | load/store/RMW | 有 | 是，且更强 | 所有 seq_cst 操作一条全局总序 |

强度可以单边降，不能乱配。`seq_cst` store 配 `acquire` load，读到值时仍然能同步；`release` store 配 `relaxed` load，没有 synchronizes-with。配对看的是两边实际用的语义，不是名字好不好听。

## 可见性和同步分别保证什么

把这句话拆开，很多 API 选择会变清楚。

**原子性**只谈一个对象的一次操作。`relaxed` 的 `fetch_add` 不会丢更新，两个线程各加一，结果一定是加了二。它不谈 `fetch_add` 周围那些普通变量。

**可见性**谈的是：一次写的结果，另一次读能不能观察到。没有 happens-before，模型不承诺可见。硬件上它常常「过一会儿就可见」，那是缓存一致性最终会把 store buffer 冲出去，不是 C++ 给你的保证，也不是你可以写进 assert 的东西。

**同步**谈的是：用一次原子操作（或一把锁），把**一整批**先前的写打包，让另一侧的一整批读都能看见。release/acquire 的价值不在 `flag` 本身从 false 变 true，而在 `flag` 旁边那些普通数据一起过桥。

所以会看到这种分工：

```text
payload   —— 普通对象，只负责数据
flag      —— 原子对象，只负责同步
```

把 payload 也改成 `atomic` 并全部 `seq_cst`，能跑，但你买到的是多余的屏障，不是更多正确性。正确性来自那座桥，不来自每个字段都原子。

反过来：只把 payload 改成 `atomic`、用 `relaxed` 读写，没有 flag 上的 acquire/release，仍然没有「整包数据一起可见」的保证。原子字段各自有修改序，字段和字段之间没有。

## 几段会写错的代码

### 自旋锁

```cpp
class SpinLock {
    std::atomic<bool> locked{false};

public:
    void lock() {
        while (locked.exchange(true, std::memory_order_acquire)) {
            while (locked.load(std::memory_order_relaxed)) {
                // pause / yield
            }
        }
    }

    void unlock() {
        locked.store(false, std::memory_order_release);
    }
};
```

`lock` 的 `exchange` 必须是 acquire：拿到锁之后读临界区，要看见上一个持有者写进去的东西。`unlock` 必须是 release：离开临界区之前的写，要发布给下一个 acquire。内层那次 `relaxed` load 只是在自旋里偷看「是不是还锁着」，不进入临界区，不必建桥。

写成两边 `relaxed`，临界区里的普通数据就是数据竞争。

### 把工作交给另一个线程

```cpp
std::atomic<Task*> mailbox{nullptr};

// 线程 A
auto* t = new Task{...};
mailbox.store(t, std::memory_order_release);

// 线程 B
if (auto* t = mailbox.load(std::memory_order_acquire)) {
    t->run();
}
```

指针本身的传递是原子的；`Task` 内部字段能看见，是因为 release/acquire 把 `new Task` 时的那些写带了过去。这里用 `relaxed` 存指针，`Task` 的构造对 B 不可见。

### 引用计数

```cpp
void add_ref() {
    refs.fetch_add(1, std::memory_order_relaxed);
}

void release() {
    if (refs.fetch_sub(1, std::memory_order_acq_rel) == 1) {
        delete this;
    }
}
```

加引用不发布任何新数据，`relaxed` 够。最后一次减一要两件事同时成立：

- acquire：看见其他线程在对象活着时写下的东西，`delete` 才安全
- release：让「引用已经掉到零」对还在路上的访问可见（和其它同步原语配合时尤其重要）

所以最后一次用 `acq_rel`（或 `fetch_sub(release)` 再补一条 `atomic_thread_fence(acquire)`）。每一次减都用 `seq_cst` 能正确，只是更贵。

### 错误的「双重检查锁定」

```cpp
// 错
if (!p.load(std::memory_order_relaxed)) {
    std::lock_guard lk(m);
    if (!p.load(std::memory_order_relaxed)) {
        p.store(new Singleton, std::memory_order_relaxed);
    }
}
Singleton* s = p.load(std::memory_order_relaxed);
s->use();  // 可能看见没构造完的对象
```

外层读必须是 acquire，里层发布必须是 release：

```cpp
Singleton* s = p.load(std::memory_order_acquire);
if (!s) {
    std::lock_guard lk(m);
    s = p.load(std::memory_order_relaxed);  // 锁内，已经有互斥同步
    if (!s) {
        s = new Singleton;
        p.store(s, std::memory_order_release);
    }
}
s->use();
```

锁本身已经在内外之间建了 happens-before，所以锁内第二次 load 用 `relaxed` 没问题。锁外第一次 load 没有锁，必须自己 acquire。

## fence：把 order 从操作上揭下来

有时原子操作本身只想 `relaxed`，同步放在旁边：

```cpp
data = 42;
std::atomic_thread_fence(std::memory_order_release);
flag.store(true, std::memory_order_relaxed);

// 另一侧
if (flag.load(std::memory_order_relaxed)) {
    std::atomic_thread_fence(std::memory_order_acquire);
    use(data);
}
```

fence 的 release/acquire 和原子上的 release/acquire 遵循同一套配对规则，只是屏障是「插在这个位置」，不是「附在这次 load/store 上」。读起来更绕，也更容易漏。能写在 `store(release)` / `load(acquire)` 上就写在操作上。

`atomic_signal_fence` 只约束编译器，不发 CPU 屏障，给信号处理函数用，不是给另一个核用的。

## 硬件上大概对应什么

模型是抽象的，但知道常见架构怎么降，有助于判断「我是不是在为并不存在的乱序付钱」。

**x86 / x64** 的普通 load/store 已经是相当强的：store-store、load-load、load-store 大致保序，弱的是 store 后面的 load（store buffer）。所以：

- `relaxed` / `acquire` load 往往就是一条普通 `mov`
- `release` store 往往也是普通 `mov`
- `seq_cst` store 需要锁前缀或 `mfence`
- RMW（`lock xadd` 等）本身就是全屏障

在 x86 上把 acquire/release 写成 `seq_cst`，很多时候慢在 store，不在 load。正确性上它们不是一回事，Dekker 那种握手在 x86 上用 acq/rel 仍然是错的——x86 的 store-load 重排正好允许 `r1==0 && r2==0`。

**ARM / RISC-V / POWER** 弱得多，load 和 store 都可以跨。acquire 是 `ldar` / `lwsync` 一类，release 是 `stlr` / `lwsync`，`seq_cst` 往往两边都升级。在这些架构上乱写 `seq_cst`，账单更明显。

不要按「我在 x86 上跑过」来选 memory order。按模型选，再让编译器在对应架构上发最便宜的指令。

## 怎么选

按问题选，不按「越强越安心」选。

1. 这段原子操作旁边有没有普通数据要一起看见？没有，考虑 `relaxed`。
2. 有，而且方向是「我写完数据，再举手」：store `release`，对面 load `acquire`。
3. 一次 RMW 既要看见前人、又要发布给后人：`acq_rel`。
4. 两个线程对称地各写各的标志、再读对方，又无法改成单向发布：`seq_cst`。
5. 临界区很大、逻辑很普通：用 `std::mutex`。它的 lock/unlock 就是 acquire/release，还能避免你自己把桥搭错。

无锁不是更快的锁，是把同步粒度从「一段代码」收成「一个原子变量」。粒度收得越细，memory order 就要写得越准。写不准的无锁，比一把粗锁更慢，也更错。

## 收住

C++ 的内存模型不保证「写完别人立刻看见」，它保证的是：

- 没有数据竞争，否则整个程序没有意义
- 有 happens-before，则先发生的那一侧的写，后发生的那一侧一定看得见
- happens-before 靠同一线程的程序顺序，加上跨线程的 synchronizes-with
- synchronizes-with 来自锁、线程起停，以及**配对成功的** atomic acquire/release（或更强的 `seq_cst`）

`memory_order` 就是在控制最后这一项：这座桥要不要、用哪种强度。可见性是过桥之后的结果；先把桥搭对，再谈省几道屏障。
