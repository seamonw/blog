---
title: 跳表赢的不是好写：用 Go 实现 skiplist
date: 2026-09-15 20:10:00
categories: Golang
tags:
	- golang
	- skiplist
	- 数据结构
---

「Redis 的 zset 用跳表而不用红黑树，因为跳表好写。」这句话流传很广，只对三分之一。好写是真的——红黑树的 delete fixup 有五种情况，跳表的删除是十行循环。但如果只是好写，Redis 早该换成 B 树或 AVL，反正只写一次。真正决定选择的是另外两件事：

- **范围扫描是免费的。** 跳表的第 0 层就是一条有序链表，`ZRANGE 1000 2000` 定位一次然后一直走 `next`。红黑树要维护迭代器栈或者 parent 指针，`successor()` 每一步都要向上爬再向下走，缓存局部性也差。
- **插入只改动局部。** 跳表插入期望只修改 `1/(1-p)` 个前驱指针，全部在查找路径上；红黑树的旋转会改动到祖先，锁的范围沿着路径往上扩散。这是为什么 RocksDB 的 memtable、Java 的 `ConcurrentSkipListMap` 都是跳表而不是并发红黑树。

代价也得先说清楚：跳表的 O(log n) 是**期望**，不是最坏。抛硬币抛砸了，它就是一条链表。本文推导这个期望从哪来、`p` 和 `maxLevel` 怎么定，然后给一份能编译的 Go 实现（`Set` / `Get` / `Delete` / 范围迭代），最后拆三个在生产里真会咬人的坑。完整代码在 [seamonw/skiplist](https://github.com/seamonw/skiplist)。

## 多层链表 + 抛硬币

一条有序链表查找是 O(n)，因为你只能一步一步走。给它加一层「快车道」，每隔几个节点建一个索引，查找就先在快车道上跨大步，跨过头了再回到慢车道。层数继续叠加，就是跳表。

关键在于：**索引层不是按固定间隔建的，是抛硬币决定的。** 每个节点插入时独立掷硬币，正面继续升一层，反面停下。以概率 `p` 晋升，节点高度 `H` 的分布是 `P(H = k) = p^(k-1) * (1-p)`。

固定间隔的多级索引（比如「每 2 个节点提一个」）查找同样快，但插入一个元素要重排后面所有索引，退化成 O(n)。抛硬币的意义是把「维护平衡」这件事从确定性的结构调整，换成概率保证：每个节点的高度和别人无关，插入只影响自己和自己的前驱。没有旋转，没有再平衡。

7 个 key 的一种可能形状（为了把层画清楚，这里的高度比 `p=1/4` 的期望值偏高）：

```text
level 3   H ─────────────────────────────────────> 17 ───────────────────> nil
level 2   H ───────────────> 6 ──────────────────> 17 ───────────────────> nil
level 1   H ────> 3 ───────> 6 ────────> 12 ─────> 17 ─────────> 25 ─────> nil
level 0   H ────> 3 ───────> 6 ──> 9 ──> 12 ─────> 17 ──> 21 ──> 25 ─────> nil
```

`H` 是头结点（哨兵），高度恒为 `maxLevel`，不存数据，只是为了让「第 i 层的第一个节点」有统一入口，省掉所有「插在表头」的特殊分支。空表就是所有层的 `H.next[i] == nil`，不需要单独判断。

查找 21 的路径：

```text
level 3   H ══> 17        17.next[3] = nil，下降
level 2         17        17.next[2] = nil，下降
level 1         17 ──✗ 25 25 > 21，不能走，下降
level 0         17 ══> 21 命中
```

规则只有一条：**在当前层，只要后继的 key 严格小于目标就前进，否则下降一层。** 走到第 0 层时，手里的 `x` 一定是最后一个 `key < target` 的节点，所以候选答案只可能是 `x.next[0]`，比较一次就知道命不命中。

插入和删除都复用这次查找，只多做一件事：**下降前把当前层的 `x` 记到 `prev[i]` 里**。走完一遍，`prev[0..level-1]` 就是每一层的插入点。插入是在 `0..height-1` 这些层上做链表插入；删除是在这些层上做链表摘除。没有第二次遍历。

### 期望高度与期望复杂度

单个节点的期望高度 `E[H] = 1/(1-p)`。`p=1/2` 是 2，`p=1/4` 是 1.33 —— 这直接就是平均每个节点的指针数。

`n` 个节点里最高的那个，期望高度约 `log_{1/p} n`。所以查找从大约 `log_{1/p} n` 层开始往下走。每一层往前走多少步？反向分析：从目标节点往左上方爬，每一步「向上」的概率是 `p`，「向左」的概率是 `1-p`，要爬 `log_{1/p} n` 层，期望总步数是

```text
C(p) = (1/p) · log_{1/p} n
```

代进去：`p=1/2` 得 `2·log₂n`，`p=1/4` 得 `4·log₄n = 2·log₂n`。**两者的期望比较次数一样**，但 `p=1/4` 的指针数只有前者的 2/3。这就是 Pugh 原论文推荐 `1/4` 的理由，也是 Redis、LevelDB 都取 `1/4` 的原因。理论最优在 `p = 1/e ≈ 0.368`（约 `1.88·log₂n`），但 `1/4` 能用移位实现、方差更小，工程上更划算。

`maxLevel` 就是 `log_{1/p} N`，`N` 是你预期的最大元素数：

| p | 目标 N | maxLevel | 平均指针/节点 |
| --- | --- | --- | --- |
| 1/4 | 1.6e7 (4¹²) | 12（LevelDB 取值） | 1.33 |
| 1/4 | 1.8e19 (4³²) | 32（Redis 取值） | 1.33 |
| 1/2 | 4.3e9 (2³²) | 32 | 2.0 |

`maxLevel` 设大了只是浪费头结点的一点内存（`maxLevel` 个指针，32 层也就 256 字节），设小了会实打实变慢：`maxLevel` 封顶后，顶层节点数约 `n·p^(maxLevel-1)`，这些节点必须被线性扫过。`p=1/4`、`maxLevel=8`、`n=10⁶` 时顶层有约 61 个节点，查找从「跨 61 步」开始 —— 已经不是 O(log n) 了。宁可设大。

和平衡树摊开对比：

| | 红黑树 | 跳表 |
| --- | --- | --- |
| 查找 | O(log n) 最坏 | O(log n) 期望，最坏 O(n) |
| 插入的结构改动 | 旋转，改动涉及祖先节点 | 期望改 `1/(1-p)` 个前驱指针，全在查找路径上 |
| 范围扫描 | 中序遍历，需要栈或 parent 指针 | 第 0 层即有序链表，一次 `next` |
| 并发 | 旋转使锁沿路径上移 | 逐层 CAS，或单写多读免锁 |
| 指针/节点 | 2（+parent 是 3）+ 色位 | 1.33（p=1/4） |
| 删除实现 | fixup 五种情况 | 逐层摘链，十行 |

跳表输在「最坏」和「随机源依赖」，赢在范围扫描、并发友好和实现可审查。在有序集合这个场景里，后三条的权重更高。

## 一份能跑的实现

用泛型约束 `cmp.Ordered`，键类型交给调用方。需要 Go 1.23（`iter` 包）；去掉最后那个迭代器函数则 Go 1.22 即可（`math/rand/v2`）。

```go
package skiplist

import (
	"cmp"
	"iter"
	"math/rand/v2"
)

const (
	maxLevel = 32              // p=1/4 时覆盖到 4^32 个元素，够任何进程内场景
	pThresh  = uint32(1) << 30 // 2^32 / 4，即晋升概率 p = 1/4
)

type node[K cmp.Ordered, V any] struct {
	key  K
	val  V
	next []*node[K, V] // len(next) 就是这个节点的高度
}

type List[K cmp.Ordered, V any] struct {
	head  *node[K, V] // 哨兵，高度恒为 maxLevel，不存数据
	level int         // 当前实际用到的最高层数，1-based
	n     int
	rnd   func() uint32 // 抽出来是为了测试时能注入确定性随机源
}

func New[K cmp.Ordered, V any]() *List[K, V] {
	return &List[K, V]{
		head:  &node[K, V]{next: make([]*node[K, V], maxLevel)},
		level: 1,
		rnd:   rand.Uint32, // math/rand/v2 的顶层函数免锁且并发安全
	}
}

func (l *List[K, V]) Len() int { return l.n }

// randomLevel 连续抛硬币，正面（概率 p）就升一层。
func (l *List[K, V]) randomLevel() int {
	lv := 1
	for lv < maxLevel && l.rnd() < pThresh {
		lv++
	}
	return lv
}

// search 返回每一层最后一个 key 严格小于 target 的节点。
// prev[0].next[0] 就是候选答案。
func (l *List[K, V]) search(target K, prev *[maxLevel]*node[K, V]) {
	x := l.head
	for i := l.level - 1; i >= 0; i-- {
		// 只要后继严格小于目标就前进；等于或大于就下降。
		for nxt := x.next[i]; nxt != nil && nxt.key < target; nxt = x.next[i] {
			x = nxt
		}
		prev[i] = x
	}
}

func (l *List[K, V]) Get(key K) (V, bool) {
	x := l.head
	for i := l.level - 1; i >= 0; i-- {
		for nxt := x.next[i]; nxt != nil && nxt.key < key; nxt = x.next[i] {
			x = nxt
		}
	}
	if nxt := x.next[0]; nxt != nil && nxt.key == key {
		return nxt.val, true
	}
	var zero V
	return zero, false
}

// Set 是 upsert：key 已存在时只覆盖 value，不动结构也不改高度。
func (l *List[K, V]) Set(key K, val V) {
	var prev [maxLevel]*node[K, V] // 值数组，不逃逸，留在栈上
	l.search(key, &prev)

	if nxt := prev[0].next[0]; nxt != nil && nxt.key == key {
		nxt.val = val
		return
	}

	lv := l.randomLevel()
	if lv > l.level {
		// 新高度超过当前最高层，多出来的层此前从未被访问，前驱只能是哨兵。
		for i := l.level; i < lv; i++ {
			prev[i] = l.head
		}
		l.level = lv
	}

	x := &node[K, V]{key: key, val: val, next: make([]*node[K, V], lv)}
	for i := 0; i < lv; i++ {
		x.next[i] = prev[i].next[i]
		prev[i].next[i] = x
	}
	l.n++
}

func (l *List[K, V]) Delete(key K) bool {
	var prev [maxLevel]*node[K, V]
	l.search(key, &prev)

	x := prev[0].next[0]
	if x == nil || x.key != key {
		return false
	}
	// x 的高度不会超过 l.level（Set 里同步抬高过），所以 prev[i] 一定有效；
	// 且 prev[i] 是该层最后一个 < key 的节点，它在第 i 层的后继必然是 x。
	for i := 0; i < len(x.next); i++ {
		prev[i].next[i] = x.next[i]
	}
	// 顶层被删空就降层，否则每次查找都要白跑几层空转。
	for l.level > 1 && l.head.next[l.level-1] == nil {
		l.level--
	}
	l.n--
	return true
}
```

几点需要说明。

**`search` 里的双层循环写法。** 内层条件必须是 `nxt.key < target`（严格小于），不能是 `<=`。取 `<=` 时 `prev[0]` 会停在 key 本身，`Delete` 就摘不到目标。这是手写跳表最常见的一处 off-by-one。

**`prev` 用值数组而不是切片。** `[32]*node` 是 256 字节，`search` 只读它、不把它存进任何堆对象，逃逸分析能把它留在栈上：整条插入路径只有节点自身和它的 `next` 两次分配，`-benchmem` 下是 `2 allocs/op、58 B/op`。写成 `make([]*node, maxLevel)` 其实也能留在栈上（长度是编译期常量，实测同样 2 allocs）。但一旦你把 `maxLevel` 改成可配置的实例字段，长度变成运行时值，这次 `make` 就必然上堆 —— 实测涨到 `3 allocs/op、314 B/op`。用固定数组是把这件事钉死，不依赖编译器某一版的推断。

**`l.level` 只涨不主动缩（除了 `Delete` 里的收尾）。** 抬高发生在 `lv > l.level` 时，此时 `l.level` 到 `lv-1` 这些层在 `search` 里根本没被遍历过，`prev` 是零值，必须补成 `head`。漏掉这个循环会写出 nil 解引用。

**upsert 还是「已存在就不插」是语义选择，不是实现细节。** 上面选 upsert，因为 `Set` 这个名字承诺覆盖。如果要的是「多重集」（允许重复 key，比如 Redis zset 按 score 排序、score 可重复），那么内层条件要变成「score 相同时再比 member」，用复合键排序，而不是允许同 key 多节点 —— 后者会让 `Delete` 不知道该摘哪一个。

**随机源。** `math/rand/v2` 的顶层 `rand.Uint32()` 用的是 per-M 的廉价状态，无锁且并发安全；`math/rand`（v1）的顶层函数背后是一把全局互斥锁，在插入热路径上会成为瓶颈。反过来，`*rand.Rand`（无论 v1 v2）**不是**并发安全的，如果你把 `rnd` 换成 `rand.New(rand.NewPCG(1, 2)).Uint32` 做确定性测试，这个 List 就只能单 goroutine 用。

### 范围扫描

跳表的主场。定位一次 `O(log n)`，之后每个元素 `O(1)`：

```go
// Range 从第一个 key >= start 的元素开始正序遍历，f 返回 false 则停止。
func (l *List[K, V]) Range(start K, f func(K, V) bool) {
	x := l.head
	for i := l.level - 1; i >= 0; i-- {
		for nxt := x.next[i]; nxt != nil && nxt.key < start; nxt = x.next[i] {
			x = nxt
		}
	}
	for cur := x.next[0]; cur != nil; cur = cur.next[0] {
		if !f(cur.key, cur.val) {
			return
		}
	}
}

// From 适配 Go 1.23 的 range-over-func：for k, v := range list.From(100) { ... }
func (l *List[K, V]) From(start K) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) { l.Range(start, yield) }
}
```

闭区间 `[lo, hi]` 就在 `f` 里判 `k > hi` 返回 `false`，不需要另写一个方法。`iter.Seq2` 的签名和 `Range` 的回调完全同形，所以 `From` 只是一层类型转换，`yield` 直接当 `f` 传进去。

要注意 `Range(start)` 的语义是 `>= start`，`start` 取键类型零值即为全表遍历。对 `int` 是 0，有负数 key 时得显式传 `math.MinInt`，这类边界比看起来更容易错。反向遍历需要在节点上加 `prev` 指针（Redis 的 `zskiplistNode` 有 `backward`，只在第 0 层），本文的版本不支持。

## 三个会咬人的坑

### 1. 随机源出问题，结构悄悄退化成链表

跳表的性能保证完全押在「高度独立随机」上。这个前提被破坏时，结构不会报错，只会慢下来 —— 这类问题在压测里表现为 P99 抬升，而不是崩溃。

常见破坏方式有两种。一是**用 key 的哈希决定高度**（有人这么写是为了让重建后的结构一致），于是高度变成 key 的确定性函数，特定的 key 分布（或构造出来的输入）可以让所有节点都是 level 1，查找变成 O(n)。二是**注入了确定性种子做测试，然后忘了换回去**，比如固定种子的 PCG 在某段序列里连续给出低高度。

排查方法是直接量一下每层的节点数，理论值是 `n·p^i`：

```go
// LevelHist[i] 是第 i 层的节点数，期望约 n·p^i。偏离一个量级就说明随机源有问题。
func (l *List[K, V]) LevelHist() []int {
	h := make([]int, l.level)
	for i := range h {
		for cur := l.head.next[i]; cur != nil; cur = cur.next[i] {
			h[i]++
		}
	}
	return h
}
```

`n=100000`、`p=1/4` 时理论值是 `[100000, 25000, 6250, 1563, 391, 98, 24, 6, 1]`，上面这份实现实测跑出 `[100000, 25048, 6311, 1552, 402, 99, 21, 7, 1]`，逐层贴合。如果第 1 层只有几百个，说明 `rnd()` 几乎总返回大值 —— 检查是不是把 `pThresh` 和比较方向写反了，或者用了 `rand.Uint32() % 4 == 0` 这种在某些实现上低位质量差的写法（`math/rand/v2` 的低位没问题，但换成自己写的 LCG 就会踩）。

还要记住这是**概率**保证，不是输入无关的保证：即使随机源完好，`P(查找代价 > 3·期望值)` 不是零，只是很小。对有硬性最坏延迟要求的场景（实时交易撮合），该选 B+ 树而不是跳表。

### 2. 并发：读多写少容易，无锁删除很难

上面那份实现是**非并发安全**的，多 goroutine 同时 `Set` 会写坏指针。补救路径有三档，成本差一个量级。

**第一档：一把 `sync.RWMutex`。** `Get`/`Range` 拿读锁，`Set`/`Delete` 拿写锁。绝大多数进程内场景到这里就够了。注意 `Range` 持锁的时间等于回调的执行时间，`f` 里做 I/O 就等于把写者锁死几十毫秒 —— 这和把慢任务塞进任务池堵住 worker 是同一类错误，见 [goroutine 很轻，所以你更需要任务池](/blog/2026/09/14/golang-worker-pool/)。要长时间扫描就先把 key 收集出来再放锁。

**第二档：单写多读，无锁。** RocksDB 的 `InlineSkipList`、LevelDB 的 memtable 走这条路，前提是**只插不删**（删除在 LSM 里是写一条 tombstone，也是插入）。少了删除，无锁就简单得多：

```go
// 只插不删、单写多读的骨架。level 的原子化和 Get 省略。
type cnode[K cmp.Ordered, V any] struct {
	key  K
	val  V
	next []atomic.Pointer[cnode[K, V]]
}

// insert 只能由唯一的 writer goroutine 调用；Get 可以任意并发。
func (l *CList[K, V]) insert(key K, val V) {
	var prev [maxLevel]*cnode[K, V]
	l.searchAtomic(key, &prev) // 与单机版同形，只是用 next[i].Load() 读

	lv := l.randomLevel()
	x := &cnode[K, V]{key: key, val: val, next: make([]atomic.Pointer[cnode[K, V]], lv)}
	for i := 0; i < lv; i++ {
		x.next[i].Store(prev[i].next[i].Load()) // 先把自己的出边配好
	}
	// 必须自底向上发布：第 0 层链上的那一刻，这个 key 才算存在。
	for i := 0; i < lv; i++ {
		prev[i].next[i].Store(x)
	}
}
```

顺序是这里唯一的要点。查找总要下降到第 0 层才做相等判断，所以只有第 0 层的链接是「可见性事件」，高层纯粹是加速索引。若先链高层，读者可能在第 2 层走到 `x`，再下降到第 0 层时 `x` 的第 0 层前驱还没指向它 —— 于是这个 key 在部分读者眼里存在、在部分读者眼里不存在。反过来（自底向上）最坏只是某些读者暂时少走一层索引，答案始终正确。Go 的 `atomic.Pointer` 提供的是顺序一致语义，不需要手动写 acquire/release；换成 C++ 得自己挑 memory order，见 [C++ memory order：多线程下的可见性与同步](/blog/2026/08/28/cpp-memory-order/)。

**第三档：完全无锁，含删除。** 这才是难的部分。单纯用 CAS 摘链会丢数据，经典场景是这样：线程 A 要删 `x`，做 `CAS(&prev.next, x, x.next)`；线程 B 同时要在 `x` 之后插 `m`，做 `CAS(&x.next, oldNext, m)`。两个 CAS 各自的期望值都成立，都会成功 —— 但 A 摘掉 `x` 时读到的是 `oldNext`，`m` 挂在了已经被摘出链表的 `x` 后面，从表里凭空消失，而两个线程都收到了成功。

Harris/Michael 的解法是**两阶段删除**：先把 `x.next` 打上删除标记（逻辑删除），此后任何针对 `x.next` 的 CAS 都会失败，插入者被迫重新查找；只有标记成功之后才做物理摘链，摘不干净就交给后续遍历的线程顺手清理。Java 的 `ConcurrentSkipListMap` 用一个特殊的 marker 节点代替指针标记位（Java 也没法在指针里塞标记位）。Go 里同理：`atomic.Pointer` 不给你低位，要么加一个 `marked atomic.Bool` 字段配合 CAS 重试，要么插 marker 节点。

结论很直接：**不要为了「跳表天生适合无锁」就去手写第三档。** 除非你在写存储引擎且 profile 证明 `RWMutex` 是瓶颈，否则第一档或第二档。真要并发有序 map，先看现成实现。

### 3. 迭代期间改结构：Go 不会崩，但结果是错的

C++ 里迭代时删元素是 use-after-free，多半直接崩，问题暴露得早。Go 有 GC，被摘掉的节点内存还在、`next` 指针还在，遍历能一路走下去 —— 于是错误变成静默的脏数据。

具体三种表现：

- **在 `f` 里删当前 key**：节点被摘链，但它的 `next[0]` 仍指向原后继，遍历「意外地」正确。这是最危险的一种，因为它在测试里过了，给人一种可以边扫边删的错觉。
- **在 `f` 里删靠后的 key**：如果那个节点已经被读到过没事；还没走到就被摘掉，也没事（走不到）。但如果删的是当前节点的直接后继，而当前节点自己已经被别的 goroutine 摘掉，你就会走进一段被摘掉的旧链，读到早已删除的数据。
- **在 `f` 里插入更小的 key**：新元素插在扫描位置之前，永远不会被这次遍历看到。做「扫描全表并对每个元素重新入表」这类操作时必然出错。

规则写死：**`Range` 的回调里不改这个 List。** 需要边扫边改就两段式 —— 先收集 key，放掉锁（或结束遍历），再逐个操作，并对每个 key 重新确认是否还存在：

```go
var stale []K
list.Range(lo, func(k K, v V) bool {
	if k > hi {
		return false
	}
	if expired(v) {
		stale = append(stale, k)
	}
	return true
})
for _, k := range stale {
	list.Delete(k) // Delete 返回 false 说明期间已被别人删掉，不是错误
}
```

真要支持「快照式遍历」，得上 epoch 或引用计数让被删节点延迟回收，并给迭代器一个版本号 —— 复杂度直接跳一个量级，绝大多数业务不需要。

### 顺带说内存

`p=1/4` 时平均 1.33 个指针/节点，看着比红黑树（2 个指针 + 色位）省。但 Go 的 `next []*node` 带一个 24 字节的 slice header，一个 level-1 节点的实际开销是 `key + val + 24 + 8`，header 本身就比它指向的数组大。

RocksDB 的 `InlineSkipList` 把高度数组直接内联在节点尾部（C 的柔性数组），一次分配搞定。Go 里要做同样的事只能用 `unsafe` 手动布局，或者按高度分桶做 arena 预分配。**这是规模到了之后的优化，先把语义做对**——和时间轮把「桶到期才入堆」留到最后一步优化是一样的取舍，见 [时间轮算法](/blog/2026/08/28/timing-wheel/)。

另外，跳表的节点是独立分配的，范围扫描虽然是 `O(1)/元素`，缓存局部性并不比 B+ 树好。B+ 树把一整页 key 放在连续内存里，同样扫 1000 个元素，跳表要 1000 次指针跳转（大概率 cache miss），B+ 树只要几十次。所以磁盘和大内存场景是 B+ 树的地盘，跳表的优势区间是**内存中、写多、需要并发、且删除不频繁**的有序集合 —— memtable 和 zset 恰好都在这个区间里。

## 收束

跳表用「每个节点独立抛硬币决定高度」换掉了平衡树的旋转，代价是 O(log n) 从最坏保证降级为期望保证，收益是插入只改动查找路径上期望 1.33 个指针、第 0 层天然就是可顺序扫描的有序链表。`p=1/4` 和 `maxLevel = log₄N` 是经过验证的默认值，`maxLevel` 宁大勿小。

自己写的话，`search` 把每层前驱记进一个栈上数组，`Set`/`Delete`/`Range` 全复用它，一百五十行就是完整实现。真正需要小心的不是算法，是随机源的质量、并发模型的选择（默认 `RWMutex`，只插不删才考虑无锁，含删除的无锁交给现成库），以及别在迭代回调里改结构 —— Go 的 GC 会让这个错误静默通过测试。
