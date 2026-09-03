---
title: 杀不掉的 goroutine：Go context 的设计、源码与代价
date: 2026-09-02 14:50:00
categories: Golang
tags:
	- golang
	- context
	- 源码阅读
	- 并发
---

`context.Context` 现在几乎出现在每一条 Go 调用链的第一个参数上。它看起来像语言的一部分，其实是后来才长进去的：2014 年先以独立包出现，2016 年才进标准库。要读懂它，得先问它是在什么规模、什么失败经验里被逼出来的，再看它到底包了哪几类问题，最后才落到 `context.go` 里那几种很小的结构体。

本文不再做一篇 `WithTimeout` API 清单，而是沿四条线展开：

1. context 为什么出现，它为什么是一棵只能派生、不能原地修改的树；
2. 取消、deadline、请求级数据分别解决什么，又刻意不解决什么；
3. Go 1.27 的 `propagateCancel`、`cancelCtx`、`timerCtx`、`valueCtx` 到底怎么跑；
4. `WithoutCancel`、`AfterFunc` 怎么用，以及分配、Value 深度和万人扇出的真实代价。

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

`sync.WaitGroup` 只能回答「这些 goroutine 都结束了吗」，不能通知它们结束；全局 `map[requestID]chan struct{}` 能把请求和退出信号对应起来，却要自己处理锁、删除、超时和 ID 冲突；每层传一个 `quit chan struct{}` 能取消，但 deadline、取消原因、请求 ID 又会各长出一套参数。它们都能解决局部问题，缺的是跨包统一的**调用作用域协议**。

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

### 「不可变树」到底哪里不可变

常见说法是 context 是不可变树。这句话对 API 使用者成立，但不能机械理解成内部没有可变字段。

```go
parent := context.Background()
child1 := context.WithValue(parent, requestIDKey, "req-1")
child2, cancel := context.WithCancel(child1)
defer cancel()
```

每个 `With*` 都返回一个**新节点**，不会把 `parent` 原地改成另一个含义。节点创建后，父指针、key/value、deadline 不再修改；因此同一个 parent 可以安全地被不同 goroutine 同时派生，兄弟分支也不会覆盖彼此。

但取消状态当然会变：`cancelCtx.done` 会从 nil 变成 channel，`err` 会被写入，父节点的 `children` map 会增删。所谓不可变，准确说是：

- **对外语义和向上的父链不可变**；
- **内部取消状态受锁和原子操作保护，可以发生一次状态迁移**；
- **派生而非修改**让作用域天然形成层级，也避免共享一张可变配置 map。

```text
向上：Value 查询、Deadline 继承

timerCtx ──parent──▶ valueCtx ──parent──▶ cancelCtx ──parent──▶ backgroundCtx
   │                                      │
   └──────────── 取消树中的 child ◀───────┘

向下：cancelCtx.children 广播取消
```

父指针负责「向上找」，`children` 只为「向下取消」服务。这两套方向不要混成一条链。

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

Go 1.21 的 `WithoutCancel(parent)` 故意切断这条线。这里很容易记错：它只保留 Value，**不保留取消、deadline、Err 和 Cause**。返回值的 `Done()` 是 nil，`Deadline()` 返回 `ok=false`。请求已经结束、审计日志还要写完时可以用，但必须给脱离出来的任务重新设一个边界：

```go
func handle(w http.ResponseWriter, r *http.Request) {
    result := doBusiness(r.Context())
    writeResponse(w, result)

    // 错：Handler 返回后 r.Context() 会取消，异步任务可能刚启动就失败。
    // go writeAudit(r.Context(), result)

    // 对：继承 request ID / trace 等 Value，但切断请求的取消，
    // 再给后台收尾任务一个独立的 2 秒上限。
    detached := context.WithoutCancel(r.Context())
    auditCtx, cancel := context.WithTimeout(detached, 2*time.Second)
    go func() {
        defer cancel()
        _ = writeAudit(auditCtx, result)
    }()
}
```

`WithoutCancel` 不是「让 goroutine 永生」。切断以后如果不重新加 timeout，谁来回收它就又变成了你的问题。

### Go 1.21 的 AfterFunc：取消后做一次动作

过去常为「ctx 取消时关连接」单独起一个监听 goroutine：

```go
go func() {
    <-ctx.Done()
    conn.Close()
}()
```

`context.AfterFunc` 把这件事变成注册回调：

```go
stop := context.AfterFunc(ctx, func() {
    conn.Close()
})
defer stop()
```

对标准 `cancelCtx`，回调作为一个特殊子节点注册，不需要每次调用都提前起一条 goroutine 等待；取消发生时，回调才在自己的 goroutine 中执行。`stop()` 返回 true 表示成功阻止了回调启动，返回 false 可能是已经启动，也可能是之前已 stop。它**不等待**回调结束，所以回调和正常路径如果都碰同一个资源，仍要靠 `sync.Once`、锁或资源本身的幂等性协调。

一个常见用途是给无法直接接收 Context、但提供 `Close` / `Interrupt` 的旧 API 接上取消。不要在回调里做长时间阻塞工作：它虽然不堵 `cancel` 本身的业务 goroutine，但仍会制造新的后台任务和生命周期问题。

### 它刻意不解决的

- **强制杀死 goroutine。** 听不到 `Done` 的循环，cancel 了也还在跑。
- **替代函数参数。** 见上面的 Value。
- **对象的一生。** context 描述一次调用，不该塞进结构体字段当成员变量。
- **库内部写死超时。** 超时策略属于调用方。库应尊重传入的 ctx。
- **等待 goroutine 全部退出。** `CancelFunc` 只发信号，不等待完成；需要 `WaitGroup` 或 `errgroup` 收敛。

### 三个反面教材

**把 Context 存进业务 struct：**

```go
// 错：Client 活得比一次调用久，ctx 却可能早已取消。
type Client struct {
    ctx context.Context
}

func (c *Client) Fetch(id string) error {
    return fetch(c.ctx, id)
}

// 对：生命周期写进方法签名。
func (c *Client) Fetch(ctx context.Context, id string) error {
    return fetch(ctx, id)
}
```

少数框架对象本身就代表一段生命周期，可以把 Context 放字段里；普通 service/client 不属于这种情况。判断标准不是「能不能编译」，而是对象和调用是否必然同生共死。

**用 string key，再做强制类型断言：**

```go
ctx = context.WithValue(ctx, "user", User{ID: 1})
name := ctx.Value("user").(string) // 运行时 panic
```

自定义未导出 key 类型解决包间碰撞；封装 getter 并使用 comma-ok 解决类型断言。更彻底的办法仍是：必选业务数据不要进 Context。

**创建 timeout 后把 cancel 丢了：**

```go
func call(ctx context.Context) error {
    ctx, _ = context.WithTimeout(ctx, time.Hour) // 一小时内资源可能无法回收
    return downstream(ctx)
}
```

即使你确信下游十毫秒就返回，也要保存并调用 cancel。它不只是发取消通知，还是释放父子引用和 runtime timer 的资源句柄。

## 三、源码

`src/context/context.go` 对外只暴露接口和一组 `With*` 工厂。对内是几种很小的结构体，叠成一棵树。下面以本文写作时本机的 Go 1.27 源码为准。

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

两个根在旧版本里曾有过不同写法。**当前 Go 1.27 并不是“同一个自定义 int 的指针”**，而是两个动态类型不同、都内嵌 `emptyCtx` 的零字段结构体值：

```go
type emptyCtx struct{}
type backgroundCtx struct{ emptyCtx }
type todoCtx struct{ emptyCtx }

func Background() Context { return backgroundCtx{} }
func TODO() Context       { return todoCtx{} }
```

行为完全一样：永不取消、没有值和 deadline。动态类型分开，是为了 `String()` 能分别打印 `context.Background` 和 `context.TODO`。语义上，`Background` 是「这里就是起点」，`TODO` 是「我还没想好」。HTTP 服务器的入口用的是请求级 context，不是这两个。

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
    done     atomic.Value // chan struct{}，懒创建
    children map[canceler]struct{}
    err      atomic.Value // error，只写一次
    cause    error
}
```

`WithCancel` 干两件事：new 一个 `cancelCtx`，再 `propagateCancel` 把它挂到父上。

`mu` 保护 `children`、`cause` 以及创建/关闭 done 的慢路径。`done` 和 `err` 用 `atomic.Value`，是因为 `Done()` / `Err()` 远比取消频繁；源码注释直接写着：原子读大约比 mutex 快 5 倍，在紧循环里值得。`Done()` 第一次被调用才 `make(chan struct{})`。若从来没人监听，取消时直接把一个进程内复用的 `closedchan` 填进去，连 channel 都不用新建。

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

### propagateCancel：四条传播路径

`propagateCancel` 不只是「塞进 children map」。Context 是接口，父节点可能是标准库自己的，也可能是用户自定义实现。Go 1.27 按成本从低到高走四条路：

```go
func (c *cancelCtx) propagateCancel(parent Context, child canceler) {
    c.Context = parent

    done := parent.Done()
    if done == nil {
        return // 1. 父永不取消
    }

    select {
    case <-done:
        child.cancel(false, parent.Err(), Cause(parent))
        return // 2. 父已取消，孩子出生即取消
    default:
    }

    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if err := p.err.Load(); err != nil {
            child.cancel(false, err.(error), p.cause)
        } else {
            if p.children == nil {
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
        return // 3. 找到标准 cancelCtx，直接登记
    }

    if a, ok := parent.(afterFuncer); ok {
        // 4a. 让父在取消时回调 child.cancel
        stop := a.AfterFunc(func() {
            child.cancel(false, parent.Err(), Cause(parent))
        })
        c.Context = stopCtx{Context: parent, stop: stop}
        return
    }

    // 4b. 实在无法接入，只能用一条 goroutine 搭桥
    go func() {
        select {
        case <-parent.Done():
            child.cancel(false, parent.Err(), Cause(parent))
        case <-child.Done():
        }
    }()
}
```

最关键的是第 3 条。`parentCancelCtx` 不是简单的类型断言；中间可能夹着 `valueCtx`。标准库用包内私有 key `&cancelCtxKey` 调 `parent.Value(...)`，让查询穿过包装层找到最里面的 `*cancelCtx`，再检查它的 `Done()` 是否和 parent 暴露的 Done 是同一个 channel。相同才允许绕过包装直接登记；自定义 Context 如果改写了 Done，标准库不会冒险破坏它的语义。

因此标准的 `cancelCtx → valueCtx → timerCtx` 不会为每条父子边起 goroutine，只在父的 `children` map 里加一个节点。只有用户自定义 Context 既不是标准 cancelCtx 派生、也不支持 `AfterFunc` 时，才走最后那条桥接 goroutine。

父已取消与登记 child 之间还有竞态，所以第 3 条拿 `p.mu` 后再次检查 `p.err`：要么登记成功、以后被父遍历到；要么当场取消，不会落在缝里。

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
    c.cancelCtx.propagateCancel(parent, c)
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
3. **人手 cancel 最终会 `timer.Stop()`。** 实际顺序是先取消内嵌的 cancelCtx、从父节点摘除自己，再停掉还没响的 timer。请求提前结束时不留无意义的闹钟。这是超时也要 `defer cancel()` 的第二个原因：第一个是从父表摘节点，第二个是拆闹钟。

`Deadline()` 返回自己存的时间。`Err()` 到期时是 `DeadlineExceeded`，人手 cancel 时是 `Canceled`，和接口文档一致。

所谓「忘记 cancel 导致 Timer 泄漏」，更准确的说法是**资源被保留到 deadline 或父取消**，不一定永久泄漏：父的 `children` map 强引用 timerCtx，runtime timer 的回调也捕获它；它下面挂的 Value 和业务对象因此都不能提前 GC。若 timeout 是一小时而请求 10ms 就结束，这批对象会白白活一小时。显式 cancel 同时从父表删除节点并 Stop timer，让整条链尽早可回收。

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

每次 `WithValue` 包一层。Go 1.27 的内部 `value` 用 `for + type switch` 迭代向上，而不是递归压 Go 调用栈；但复杂度仍是 O(深度)。越靠近叶子的 key 越快，找根部 key 或 miss 要扫完整条链。深度通常是个位数，线性扫描比给每个节点维护一张同步 map 更便宜，也没有额外的锁。这就是文档强调「只放极少几个值」的实现理由。

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

## 四、性能代价：树不是免费的

Context 通常不是性能瓶颈，但「它只是四个方法，所以可以随便套」也不对。派生节点会逃逸到堆，timeout 还要注册 runtime timer，Value 查询按深度线性增长，取消一个超宽的父节点则会串行遍历所有孩子。

下面是可复现的核心 benchmark（完整测试还应加入不同 key 类型和 miss）：

```go
package contextbench

import (
    "context"
    "testing"
    "time"
)

func BenchmarkDerive(b *testing.B) {
    b.Run("WithCancel", func(b *testing.B) {
        b.ReportAllocs()
        for i := 0; i < b.N; i++ {
            _, cancel := context.WithCancel(context.Background())
            cancel()
        }
    })

    b.Run("WithValue", func(b *testing.B) {
        b.ReportAllocs()
        for i := 0; i < b.N; i++ {
            _ = context.WithValue(context.Background(), 0, 0)
        }
    })

    b.Run("WithTimeoutAndCancel", func(b *testing.B) {
        b.ReportAllocs()
        for i := 0; i < b.N; i++ {
            _, cancel := context.WithTimeout(context.Background(), time.Hour)
            cancel()
        }
    })
}
```

本文在 `windows/amd64`、Ryzen 7 4700G、Go 1.27 上以 `-benchmem -count=3` 实测，三次结果很接近，取中间量级：

```text
WithValue                 ≈  35 ns/op    48 B/op   1 alloc/op
WithCancel                ≈ 109 ns/op    96 B/op   2 allocs/op
WithTimeoutAndCancel      ≈ 397 ns/op   272 B/op   4 allocs/op
```

绝对数字随 CPU 和 Go 版本变，比例更值得看：`WithTimeout` 不只是多存一个 `time.Time`，还要建 `timerCtx`、取消节点和 runtime timer。不要在热循环的每一项上无脑 `WithTimeout`；通常在批次或一次 I/O 调用边界建一个即可。

### Value 深度是实打实的 O(N)

把 key 放在链根，分别套 1、10、100 层 `WithValue` 后查它：

```text
depth=1/hit-root          ≈   5 ns/op
depth=10/hit-root         ≈  46 ns/op
depth=100/hit-root        ≈ 397 ns/op
```

结果基本按深度线性增长，而且查找本身是 0 allocation。结论不是「Value 很慢」，而是它适合少量请求元数据，不适合被当成层层覆盖的配置中心。在日志热路径里每条日志查十几个深层 key，也会把这笔小账放大。

### 万人扇出的取消尖峰

另一个 benchmark 先在计时器外给同一个父节点挂 N 个 `WithCancel` 子节点，再只测一次 `parentCancel()`：

```text
children=100             ≈    7.6 µs
children=1,000           ≈     77 µs
children=10,000          ≈    2.2 ms
```

`cancelCtx.cancel` 持有父锁遍历 map，并在持锁期间逐个调用 `child.cancel`、获取孩子的锁。这保证树的状态简单一致，但取消不是并行广播算法，而是一轮串行级联。上面的测试甚至**没有**让一万个 goroutine 阻塞在 `Done` 上；真实场景还会在 close 之后瞬间制造大量 runnable goroutine，调度尖峰可能比遍历本身更明显。

这不意味着 context 不能带一万个任务，而是不要默认一棵极宽的树取消成本为 O(1)。极端扇出可以分组建中间父节点、控制同时活跃的任务数，或用 worker pool。更重要的是：取消路径不要拿业务锁等待孩子，`CancelFunc` 只负责发信号，不负责 join。

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
- **源码：** `emptyCtx` 是永不取消的根；`cancelCtx` 用关 channel 做广播、用 `children` 做级联；`timerCtx` 在这之上加一只闹钟；`valueCtx` 是向上查找的链表。`propagateCancel` 优先登记节点，只在无法接入自定义 Context 时才起桥接 goroutine。
- **代价：** 派生会分配，Value 查询是 O(深度)，父取消是 O(子树节点数)，万人同时从 Done 醒来还会形成调度尖峰。Context 足够便宜，但绝不是免费。

写业务时落到三行习惯就够：ctx 放第一个参数传到会阻塞的那一层；`WithCancel` / `WithTimeout` 的 `cancel` 用 `defer` 调掉；阻塞处听 `ctx.Done()`，或者调用同样听它的标准库 API。

## 参考

- [Go Concurrency Patterns: Context](https://go.dev/blog/context)
- [Go Concurrency Patterns: Pipelines and cancellation](https://go.dev/blog/pipelines)
- [context package documentation](https://pkg.go.dev/context)
- [Go 1.27 context.go](https://cs.opensource.google/go/go/+/refs/tags/go1.27.0:src/context/context.go)
