---
title: Go context：创作背景、要解决的问题和源码
date: 2026-09-02 14:50:00
categories: Golang
tags:
	- golang
	- context
	- 源码阅读
	- 并发
---

`context.Context` 现在几乎出现在每一条 Go 调用链的第一个参数上。它看起来像语言的一部分，其实是后来才长进去的：2014 年先以独立包出现，2016 年才进标准库。要读懂它，得先问它是在什么规模、什么失败经验里被逼出来的，再看它到底包了哪几类问题，最后才落到 `context.go` 里那几种很小的结构体。

本文按这个顺序写：创作背景、解决的问题、源码。

## 一、创作背景

### 调用树比 goroutine 更难停

Go 1.0 把「起一个并发单元」变成了一行 `go f()`。起很容易，停却没有对等的原语。操作系统里可以杀线程、杀进程；Go 里没有 `KillG`。设计意图很明确：一个 goroutine 不该被另一个 goroutine 从外面粗暴掐掉，否则锁、buffer、连接都会停在半中间。停，必须是合作的——被停的那一方自己看见信号、自己收尾。

早期代码因此长出各式各样的退出通道：

```go
func worker(quit <-chan struct{}, jobs <-chan Job) {
    for {
        select {
        case <-quit:
            return
        case job := <-jobs:
            do(job)
        }
    }
}
```

一层调用还好办。Google 内部那种请求模型不是一层：入口一个 RPC，后面扇出到鉴权、存储、广告、再扇出到更下游。客户端断开或超时的时候，要停的不是一个 `worker`，而是整棵调用树。每一层自己发明一个 `quit`、`done`、`timeout`，签名迅速膨胀，漏传一层，那一层就永远收不到停。

Sameer Ajmani 在 2014 年的文章 *Go Concurrency Patterns: Context* 里把这件事说死了：需要一个**可派生、可沿调用链传递**的值，把取消、截止时间和这次请求特有的少量数据绑在一起。它先作为 `golang.org/x/net/context` 放出，给 `net/http`、gRPC 这类库试用。Go 1.7（2016）把它搬进标准库 `context`，`net/http` 同时有了 `Request.Context()` / `WithContext`。从此「函数第一个参数叫 `ctx`」变成了社区约定，不是某个框架的私货。

后面几年是修补，不是改方向：

| 版本 | 补上的缺口 |
| --- | --- |
| Go 1.7 | 进标准库；HTTP 请求带上 context |
| Go 1.8+ | `database/sql` 等补 `QueryContext` 一类 API |
| Go 1.20 | `WithCancelCause` / `Cause`，取消可以带具体原因 |
| Go 1.21 | `WithoutCancel`、`AfterFunc`，以及带 cause 的超时 |

背景可以收成一句：**goroutine 解决了「怎么开始」，context 解决了「整棵调用树怎么一起结束」。**

### 它有意做成接口，而不是全局开关

另一种做法是进程级的 `shutdown` 通道。那解决不了「这一次请求取消了，另一次还在跑」。Google 的负载特征是海量**短命、彼此独立**的请求，每棵树自己的生命周期，树和树之间不能互相误伤。所以 context 从一开始就是**按调用派生出来的树**，不是单例。

接口做得很小，四种能力分开，好让 `net/http`、数据库驱动、自写的 `select` 用同一套信号，而不去依赖某一个具体结构体。

## 二、它解决的问题

三件事，对应接口上四个方法（取消占了 `Done` 和 `Err` 两个）。

### 1. 取消：调用方有权说停，且能传到每一层

没有 context 时，超时只打断当前函数，派生出去的工作不知道：

```go
func handle(w http.ResponseWriter, r *http.Request) {
    result := make(chan string, 1)
    go func() {
        result <- queryDB(r.URL.Query().Get("id"))
    }()

    select {
    case v := <-result:
        fmt.Fprint(w, v)
    case <-time.After(200 * time.Millisecond):
        http.Error(w, "timeout", http.StatusGatewayTimeout)
    }
}
```

客户端已经拿到 504，`queryDB` 还占着连接。问题不是「少写了一个 sleep」，是**没有一条从入口通向叶子的停工线**。

context 的约定是：谁发起这次工作，谁拿着 `cancel`；谁阻塞，谁听 `Done`。

```go
func queryDB(ctx context.Context, id string) (string, error) {
    rows, err := db.QueryContext(ctx, "SELECT name FROM users WHERE id=?", id)
    if err != nil {
        return "", err
    }
    defer rows.Close()
    return scanName(rows)
}

func handle(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithCancel(r.Context())
    defer cancel()

    name, err := queryDB(ctx, r.URL.Query().Get("id"))
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    fmt.Fprint(w, name)
}
```

`r.Context()` 在客户端断开时会被服务器取消。`QueryContext` 听同一个 `Done`。入口取消，叶子上的驱动也会尽量停掉查询。

工作循环的标准形状是 `select`：

```go
func worker(ctx context.Context, jobs <-chan Job) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case job, ok := <-jobs:
            if !ok {
                return nil
            }
            if err := job.Do(ctx); err != nil {
                return err
            }
        }
    }
}
```

取消语义有三条，漏一条就会在规模上去之后爆：

- **父取消，子全部取消。** 整棵树一起倒。
- **子取消，父不受影响。** 下游失败不该误杀兄弟分支。
- **已经取消的父上再派生，孩子立刻是取消状态。** 不能出现「父死了，子还活着」的窗口。

这就是「树」，不是「广播变量」。

### 2. 截止时间：停也可以由闹钟来按

人手 `cancel()` 适合「客户端走了」。SLA 是另一件事：这次搜索最多 300ms，到期必须回。

```go
func handleSearch(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 300*time.Millisecond)
    defer cancel()

    result, err := search(ctx, r.URL.Query().Get("q"))
    if errors.Is(err, context.DeadlineExceeded) {
        http.Error(w, "timeout", http.StatusGatewayTimeout)
        return
    }
    if errors.Is(err, context.Canceled) {
        return
    }
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    json.NewEncoder(w).Encode(result)
}

func search(ctx context.Context, q string) (*Result, error) {
    local, err := searchLocal(ctx, q)
    if err != nil {
        return nil, err
    }
    remote, err := searchRemote(ctx, q)
    if err != nil {
        return nil, err
    }
    return merge(local, remote), nil
}
```

超时只在入口设一次。`searchLocal`、`searchRemote`、`http.NewRequestWithContext` 看见的是同一条 deadline。某层再套一个更短的 `WithTimeout`，只能**收紧**自己这一段，不能把父的截止时间改松——更早的那个一定会先响。

`Deadline()` 留给还要自己排期的代码：还剩 20ms 就别再发一个已知要 50ms 的下游。

`WithTimeout` 是 `WithDeadline(parent, time.Now().Add(d))` 的语法糖。到期后 `Err()` 是 `DeadlineExceeded`，和人手取消的 `Canceled` 分开，调用方可以回不同的 HTTP 状态码。

### 3. 请求范围内的值：不污染每一个函数签名

一次请求里，日志和 tracing 几乎每一层都要用请求 ID，但不该把 `reqID string` 加进几十个函数签名。`WithValue` 把这类**与这次请求同生共死、又不是业务入参**的数据挂在链上。

```go
type ctxKey struct{}

var requestIDKey = ctxKey{}

func withRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}

func requestIDFrom(ctx context.Context) string {
    id, _ := ctx.Value(requestIDKey).(string)
    return id
}
```

key 用未导出类型，避免和别的包的 `"requestID"` 字符串撞车。

这里有一条硬边界，是 context 被误用最多的地方：

```go
// 不要：用 Value 传业务参数
ctx = context.WithValue(ctx, "page", 2)
users := ListUsers(ctx)

// 要：签名里看得见
users := ListUsers(ctx, ListUsersQuery{Page: 2})
```

适合放的：请求 ID、trace span、已鉴权用户的只读身份。不适合放的：分页、过滤器、可选配置——那些该出现在参数列表里，让编译器和测试抓住。

### 一张图串起来

```text
r.Context()                      客户端断开，整棵树倒
 └─ WithValue(requestID)         不产生新的取消点，信号穿过它
     └─ WithTimeout(300ms)       到期，这一枝自己倒
         ├─ searchLocal(ctx)
         └─ WithTimeout(100ms)   只能更短，不能更长
             └─ searchRemote(ctx)
```

一个请求一棵树。两棵树互不打扰。`WithValue` 也是子节点，但不登记到父的「孩子表」里，因为它不引入新的取消点。

Go 1.21 的 `WithoutCancel(parent)` 故意切断这条线：请求已经结束，审计日志还要写完。切断取消，保留 deadline 和 value。用之前要想清楚，你是在做收尾，还是在掩盖一次忘记 `cancel`。

### 它刻意不解决的

- **强制杀死 goroutine。** 听不到 `Done` 的循环，cancel 了也还在跑。
- **替代函数参数。** 见上面的 Value。
- **对象的一生。** context 描述一次调用，不该塞进结构体字段当成员变量。
- **库内部写死超时。** 超时策略属于调用方。库应尊重传入的 ctx。

## 三、源码

`src/context/context.go` 对外只暴露接口和一组 `With*` 工厂。对内是几种很小的结构体，叠成一棵树。读它们比读文档更能记住行为。下面按 Go 1.22 附近的结构说，字段名后续小版本可能微调，算法没有变。

### 接口：四个方法，没有第五个

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

| 方法 | 实现上真正做的事 |
| --- | --- |
| `Done()` | 返回一个在取消时**关闭**的 channel。关闭是 Go 里唯一内建的广播：所有 `select` 同时被唤醒 |
| `Err()` | channel 关掉之后才能读到原因；关掉前返回 `nil` |
| `Deadline()` | 没有闹钟的节点直接问父；`ok == false` 表示这条链上从未设过截止时间 |
| `Value(key)` | 本节点没有就问父，一直问到根 |

没有 `Cancel()` 方法。取消权在工厂返回的那个函数上，不在接口上。这样任意拿到 `Context` 的下游都不能取消祖先，只能听。

两个根：

```go
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)

func Background() Context { return background }
func TODO() Context       { return todo }
```

行为完全一样：永不取消。语义不同——`Background` 是「这里就是起点」，`TODO` 是「我还没想好」。HTTP 服务器的入口用的是请求级 context，不是这两个。

### emptyCtx：四个空操作

```go
type emptyCtx struct{}

func (emptyCtx) Deadline() (deadline time.Time, ok bool) { return }
func (emptyCtx) Done() <-chan struct{}                   { return nil }
func (emptyCtx) Err() error                              { return nil }
func (emptyCtx) Value(key any) any                       { return nil }
```

`Done` 返回 `nil` channel。`select` 里对 `nil` channel 的 case 永远不会命中，等价于「这条退出路径不存在」。

根不会被取消，忘了 `cancel` 的节点如果挂在 `Background()` 上，就是真泄漏。挂在 `r.Context()` 上时，请求结束服务器会取消父节点，孩子通常会被带走——但那是运气，不是借口。`defer cancel()` 仍然是从父的孩子表里摘掉自己。

### canceler：孩子不必是同一种结构体

```go
type canceler interface {
    cancel(removeFromParent bool, err, cause error)
    Done() <-chan struct{}
}
```

`cancelCtx` 和 `timerCtx` 都实现它。父的孩子表用接口存，超时节点和普通取消节点可以挂在同一棵树上。`valueCtx` **不**实现这个接口，所以不进孩子表。

### cancelCtx：广播和级联都在这里

简化后：

```go
type cancelCtx struct {
    Context // 父

    mu       sync.Mutex
    done     atomic.Value // 懒创建的 chan struct{}
    children map[canceler]struct{}
    err      error
    cause    error
}
```

`WithCancel` 干两件事：new 一个 `cancelCtx`，再 `propagateCancel` 把它挂到父上。

`Done()` 第一次被调用才 `make(chan struct{})`，放进 `atomic.Value`。取消之前没有人听，就不分配 channel。

取消函数可以重入、可以并发。核心在 `cancel`：

```text
1. 已经有 err 了就返回（第二次 cancel 是空操作）
2. 记下 err、cause
3. 关闭 done channel  —— 所有阻塞在 Done 上的 goroutine 被唤醒
4. 遍历 children，对每个孩子 cancel(false, err, cause)
5. children = nil
6. 若 removeFromParent，从父的孩子表里删掉自己
```

第 4 步的 `removeFromParent=false`：整棵子树正在被拆，不必每个孙子再回头改祖父的 map，祖父马上就会把自己的表丢掉。

`propagateCancel` 处理挂到父上的两种时序：

- 父**还没**取消：加锁，放进 `parent.children`
- 父**已经**取消：立刻 `child.cancel(...)`，不入表

所以「先 cancel 父，再 WithCancel」不会漏：孩子一出生就是取消状态。

闭包返回给调用方的那个 `cancel`，内部就是 `c.cancel(true, Canceled, cause)`。`true` 表示从父表摘掉自己——这就是 `defer cancel()` 必须存在的原因：请求正常结束时，父可能还活着（例如 `Background`），不摘就是泄漏。

`WithCancelCause` 只是让第 2 步的 `cause` 变成你传入的 error。`Err()` 对人为取消仍然返回 `Canceled`，方便 `errors.Is`；具体原因走 `context.Cause(ctx)`。

### timerCtx：cancelCtx 加一只闹钟

```go
type timerCtx struct {
    cancelCtx
    timer    *time.Timer
    deadline time.Time
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // 父更早到期，自己不必再设 timer
        return WithCancel(parent)
    }
    c := &timerCtx{deadline: d}
    c.Context = parent
    propagateCancel(parent, c)
    dur := time.Until(d)
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded, nil)
        return c, func() { c.cancel(false, Canceled, nil) }
    }
    c.timer = time.AfterFunc(dur, func() {
        c.cancel(true, DeadlineExceeded, nil)
    })
    return c, func() { c.cancel(true, Canceled, nil) }
}
```

三处值得对着看：

1. **父的 deadline 更早，就退化成 `WithCancel`。** 闹钟只需要一只，挂在真正最早的那层。
2. **已经过期，创建当下就 cancel。** 不会出现「WithTimeout(0) 还跑一会儿」。
3. **人手 cancel 时先 `timer.Stop()`。** 请求提前结束，要把还没响的 timer 从运行时堆上摘掉。这是超时也要 `defer cancel()` 的第二个原因：第一个是从父表摘节点，第二个是拆闹钟。

`Deadline()` 返回自己存的时间。`Err()` 到期时是 `DeadlineExceeded`，人手 cancel 时是 `Canceled`，和接口文档一致。

### valueCtx：向上的链表，不是 map

```go
type valueCtx struct {
    Context
    key, val any
}

func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val
    }
    return value(c.Context, key)
}
```

每次 `WithValue` 包一层。查找沿父链走，O(深度)。深度通常是个位数，线性扫描比给每个节点维护一张同步 map 更便宜，也没有额外的锁。这就是文档强调「只放极少几个值」的实现理由。

`Done` / `Err` / `Deadline` 全部委托给内嵌的父。取消信号穿过这一层，就像它不存在。

key 的相等用 Go 的 `==`。两个包各自 `type key struct{}` 的零值不相等，这是推荐未导出类型的原因。字符串 `"requestID"` 会撞。

### 一次真实调用在内存里长什么样

```go
ctx := r.Context()                             // 服务器的 cancelCtx
ctx = context.WithValue(ctx, requestIDKey, id) // valueCtx
ctx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
defer cancel()
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := http.DefaultClient.Do(req)
```

```text
timerCtx  deadline=now+300ms  timer=AfterFunc
  └─ cancelCtx
       children: 可能还有别的下游
       done: chan struct{}
       Context ──▶ valueCtx {key: requestIDKey, val: "abc"}
                      Context ──▶ 请求级 cancelCtx
                                     ↑ 客户端断开从这里灌进来
```

`http.Client` 的 RoundTrip 在 `select` 里听最外层 `Done`。三件事任意一件都会让 `Do` 返回：定时器响、人手 `cancel`、客户端断开沿树向下灌。`Value(requestIDKey)` 从 timer 走到 value 那一层就返回 `"abc"`，不用改 `Do` 的签名。

### 实现加在用法上的几条约束

这些不是风格问题，是结构体自己长出来的：

| 用法 | 对应的实现原因 |
| --- | --- |
| 不要传 `nil` Context | 工厂和标准库大量解引用，没有空值检查 |
| `defer cancel()` | 从父的 `children` 摘自己；有 timer 还要 `Stop` |
| Value 只放很少的数据 | 链表查找，不是哈希表 |
| 已取消的 ctx 上再 `WithCancel`，孩子立刻死 | `propagateCancel` 发现父已有 `err` 就当场 cancel |
| 库不要写死 `WithTimeout(3s)` | 无法放宽父的 deadline，只会让调用方的 SLA 更难理解 |
| 循环里偶而看一眼 `ctx.Err()` | 没有抢占式取消，CPU 密集路径必须自己合作 |

## 收住

context 不是语法糖，是对一种失败经验的标准化。

- **背景：** goroutine 让「开始」变便宜之后，扇出型 RPC 需要一种合作式的、按请求隔离的「结束」。2014 年以独立包出现，1.7 进标准库，变成调用约定。
- **问题：** 取消要沿树向下传、截止时间只能收紧不能放宽、请求级的少量数据不要污染签名。三件事一个接口。
- **源码：** `emptyCtx` 是永不取消的根；`cancelCtx` 用关 channel 做广播、用 `children` 做级联；`timerCtx` 在这之上加一只闹钟；`valueCtx` 是向上查找的链表。树的边就是一次 `With*`。

写业务时落到三行习惯就够：ctx 放第一个参数传到会阻塞的那一层；`WithCancel` / `WithTimeout` 的 `cancel` 用 `defer` 调掉；阻塞处听 `ctx.Done()`，或者调用同样听它的标准库 API。
