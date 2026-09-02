---
title: Go context：从取消信号到实现原理
date: 2026-09-02 14:50:00
categories: Golang
tags:
	- golang
	- context
	- 并发
---

Goroutine 很容易起，停下来却不容易。一个 HTTP 请求超时了，后面的数据库查询、下游 RPC、还在算的 goroutine 如果不知道「可以走了」，就会继续占着连接、占着内存，直到它们自己结束——或者永远不结束。

`context.Context` 就是 Go 给这件事准备的标准答案：把**取消、超时、请求范围内的少量数据**放进一个值，顺着调用链往下传。本文从「没有它会怎样」开始，先把用法写对，再打开 `context` 包看取消树和四种实现。

## 先看一个会漏的 goroutine

假设处理一次请求要查库，你随手起一个 goroutine：

```go
func handle(w http.ResponseWriter, r *http.Request) {
    result := make(chan string, 1)
    go func() {
        result <- queryDB(r.URL.Query().Get("id")) // 可能很慢
    }()

    select {
    case v := <-result:
        fmt.Fprint(w, v)
    case <-time.After(200 * time.Millisecond):
        http.Error(w, "timeout", http.StatusGatewayTimeout)
    }
}
```

客户端已经收到 504 了，`queryDB` 还在跑。连接池里的连接、数据库会话、后面如果再调 RPC，全都不知道这次请求已经没人要了。超时只打断了 `handle`，没有打断它派生出去的工作。

问题可以概括成三句：

- **谁有权说停？** 调用方，不是被调用方
- **停的消息怎么传到每一层？** 不能给每个函数都加一个 `quit chan`
- **停之外还要带点什么？** 截止时间、请求 ID，而且只在这一次请求里有效

`context` 把这三件事收成一个参数。约定是：它几乎总是函数的**第一个**参数，名字就叫 `ctx`。

## Context 是一个很小的接口

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

四个方法各管一件事：

| 方法 | 作用 |
| --- | --- |
| `Done()` | 返回一个 channel，被取消或到期时关闭。在 `select` 里听它 |
| `Err()` | `Done` 关闭之后告诉你原因：`Canceled` 或 `DeadlineExceeded` |
| `Deadline()` | 截止时间，还没设则 `ok == false` |
| `Value(key)` | 取请求范围内挂上的值，没有则 `nil` |

它**不是**容器，也不是锁。你不会在里面塞业务对象；你也不会用它替代函数参数。它是一条从入口通往叶子的控制线。

标准库里所有派生都从两个根开始：

```go
ctx := context.Background() // 真正的根，永不取消
ctx := context.TODO()       // 暂时不知道该传什么时的占位
```

两者行为一样，语义不同：`Background` 是「这里就是起点」，`TODO` 是「我还没想好，别学我这样写」。新代码能确定入口的话，用 `Background`。

## 第一步：取消

最基本的派生是 `WithCancel`。它返回一个子 context 和一个 `cancel` 函数。调用 `cancel()`，这个子 context 以及它下面再派生的全部子孙，`Done` 都会被关上。

```go
func queryDB(ctx context.Context, id string) (string, error) {
    rows, err := db.QueryContext(ctx, "SELECT name FROM users WHERE id=?", id)
    if err != nil {
        return "", err
    }
    defer rows.Close()
    // ...
    return name, nil
}

func handle(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithCancel(r.Context())
    defer cancel() // 函数返回时一定释放；重复调用是安全的

    go func() {
        select {
        case <-r.Context().Done():
            cancel() // 客户端断开，主动停下游
        case <-ctx.Done():
        }
    }()

    name, err := queryDB(ctx, r.URL.Query().Get("id"))
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    fmt.Fprint(w, name)
}
```

`database/sql` 的 `QueryContext` 会监听 `ctx.Done()`。客户端断开或你自己 `cancel()`，驱动会尽量把查询停掉。

`defer cancel()` 不是可选项。`WithCancel` 会向父节点登记自己；不 cancel，这个节点会一直挂在父的孩子表里，父一直活着时就是泄漏。及时 cancel 等于从树上摘掉自己。

工作 goroutine 怎么停？标准写法是 `select`：

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

`Done()` 在未取消时是一个一直阻塞的 channel；取消后变成已关闭的 channel。`select` 到关闭的 channel 会立刻成功。`Err()` 在关闭前返回 `nil`，关闭后返回原因。

## 第二步：超时和截止时间

取消要人去按。超时是闹钟替你按。

```go
ctx, cancel := context.WithTimeout(parent, 200*time.Millisecond)
defer cancel()
```

`WithTimeout` 是 `WithDeadline(parent, time.Now().Add(d))` 的语法糖。到期后 context 自行取消，`Err()` 是 `context.DeadlineExceeded`。

仍然要 `defer cancel()`：请求提前结束时，把还没响的定时器拆掉，避免无意义的唤醒。

一个更完整的请求处理：

```go
func handleSearch(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 300*time.Millisecond)
    defer cancel()

    result, err := search(ctx, r.URL.Query().Get("q"))
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "timeout", http.StatusGatewayTimeout)
            return
        }
        if errors.Is(err, context.Canceled) {
            return // 客户端走了，不必再写响应
        }
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
    remote, err := searchRemote(ctx, q) // HTTP client 把 ctx 传到 RoundTrip
    if err != nil {
        return nil, err
    }
    return merge(local, remote), nil
}
```

`searchLocal` 和 `searchRemote` 不用各自设超时。截止时间在入口设一次，整条链路上的 `http.NewRequestWithContext`、`QueryContext`、自己的 `select` 都会看见同一个 deadline。某层再套一个更短的 `WithTimeout`，只收紧自己这一段，不会把父的截止时间改松。

`Deadline()` 给还需要自己排期的代码用：

```go
deadline, ok := ctx.Deadline()
if ok {
    // 给下游留 20ms 收尾
    if time.Until(deadline) < 20*time.Millisecond {
        return context.DeadlineExceeded
    }
}
```

## 第三步：请求范围内的值

`WithValue` 在链上挂一对 key/value，不改变取消和超时：

```go
type ctxKey string

const requestIDKey ctxKey = "requestID"

func withRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}

func requestIDFrom(ctx context.Context) string {
    id, _ := ctx.Value(requestIDKey).(string)
    return id
}
```

key 必须是可比较的。用自定义的未导出类型，避免和别人的 `"requestID"` 字符串撞车。取值时做一次类型断言。

适合放进去的，是这次请求里**每一层都可能用到、又不想污染所有函数签名**的东西：请求 ID、trace span、登录用户的只读身份。不适合放可选业务参数——那会让函数签名撒谎，编译器帮不上忙，拼写错误要到运行时才爆。

```go
// 不要这样
ctx = context.WithValue(ctx, "page", 2)
ctx = context.WithValue(ctx, "filter", "active")
users := ListUsers(ctx)

// 这样
users := ListUsers(ctx, ListUsersQuery{Page: 2, Filter: "active"})
```

`Value` 的查找是沿父链往上走，找到第一个匹配的 key。后面会看到，这就是一个链表，不是 map。

## 取消是一棵树

每次 `WithCancel` / `WithTimeout` / `WithDeadline` 都会挂到父节点下面。父一取消，子全部取消。子取消，父不受影响。

```text
r.Context()                    客户端断开，整棵树倒
 └─ WithTimeout(300ms)         到期，只倒这一枝
     ├─ searchLocal(ctx)
     └─ WithTimeout(100ms)     更短的下游超时
         └─ searchRemote(ctx)
```

这就是 context 能「循调用链传播」的原因：它不是全局开关，是一棵按请求长出来的树。一个请求一棵树，两棵树互不打扰。

`WithValue` 也产生子节点，但这个节点**不会**登记到父的孩子表里，因为它不引入新的取消点。取消信号穿过它，继续往下传——`valueCtx` 自己的 `Done` 直接调用内嵌的父 `Done`。

Go 1.21 起还有 `WithoutCancel(parent)`：派生一个**听不见父取消**的 context，用于「请求已经结束，但审计日志还要写完」这种收尾。它切断取消，保留 deadline 和 value。要用的时候先想清楚：你是在故意让工作活过请求，还是在掩盖一次忘记 cancel。

## 打开源码：四种实现

`context` 包对外只暴露接口，对内是几种很小的结构体。读它们比读文档更能记住行为。

### emptyCtx：永不取消的根

`background` 和 `todo` 是两个 `emptyCtx` 单例。四个方法都是空操作：`Done` 返回 `nil` channel（`select` 到 `nil` channel 永远不会命中），`Err` 返回 `nil`，`Value` 返回 `nil`，没有 deadline。

根不会被取消，所以树上的泄漏最终会停在这里：你忘了 cancel 的节点，会一直挂在某个长期活着的 parent 上。HTTP 服务器里 parent 往往是这次请求的 `r.Context()`，请求结束后它会被取消，忘 cancel 的子节点通常也会被带走。进程级的 `Background()` 上挂东西，忘 cancel 就是真泄漏。

### cancelCtx：真正干活的节点

简化后的结构大致是：

```go
type cancelCtx struct {
    Context                     // 父

    mu       sync.Mutex
    done     atomic.Value       // chan struct{}，懒创建
    children map[canceler]struct{}
    err      error
    cause    error              // Go 1.20+ WithCancelCause
}
```

`Done()` 第一次调用时才 `make(chan struct{})`，放进 `atomic.Value`。取消时关掉这个 channel。所有在 `select` 里等着的 goroutine 被同时唤醒——关闭 channel 是 Go 里唯一内建的「广播」。

`cancel` 的流程可以记成四步：

```text
1. 已经取消过就返回（cancel 可重入、可并发）
2. 记下 err / cause，关闭 done
3. 遍历 children，对每个孩子再调 cancel
4. 把 children 置 nil；若需要，从父的孩子表里删掉自己
```

父在创建子的时候，如果自己还没取消，就把子放进 `children`；如果自己**已经**取消，则立刻取消这个新子，不再入表。这保证了「先 cancel 父、再 WithCancel」也不会漏。

孩子通过 `canceler` 接口挂上来，不直接依赖 `*cancelCtx`。`timerCtx` 同样实现这个接口，所以超时节点也能当别人的孩子。

`WithCancelCause` 让你可以带一个更具体的错误：

```go
ctx, cancel := context.WithCancelCause(parent)
cancel(fmt.Errorf("client gone"))
fmt.Println(context.Cause(ctx)) // client gone
fmt.Println(ctx.Err())          // context canceled
```

`Err()` 对取消仍然是 `Canceled`，方便 `errors.Is`。真正原因走 `Cause`。下游库多半只认 `Err()`，自己的代码可以多看一眼 `Cause`。

### timerCtx：在 cancelCtx 上加一只闹钟

```go
type timerCtx struct {
    *cancelCtx
    timer *time.Timer
    deadline time.Time
}
```

创建时若父的 deadline 比自己更早，就不必再设定时器，直接用父的即可——更早的那个一定会先响。否则 `time.AfterFunc` 在截止点调 `cancel`，`err` 设为 `DeadlineExceeded`。

手动 `cancel()` 会先 `timer.Stop()`，再走普通取消。这就是为什么超时后也要 `defer cancel()`：请求提前结束时把 timer 从运行时堆上摘掉。

### valueCtx：一条向上的链表

```go
type valueCtx struct {
    Context
    key, val any
}

func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

没有 map，没有哈希。每次 `WithValue` 包一层，查找是 O(深度)。这就是「只放极少几个值」的实现理由：深度通常是个位数，线性扫描比维护一张同步 map 更便宜，也更简单。

key 用指针比较（对反射类型）或值比较。两个不同包里的 `type key struct{}` 零值不相等，这是推荐未导出类型作 key 的原因。

`valueCtx` 不实现 `canceler`，不进父的 `children`。它只是一层透明包装。

## 串起来的一次调用

一次带超时和请求 ID 的下游 HTTP 调用，内部实际长这样：

```go
ctx := r.Context()                                // 服务器给的 cancelCtx
ctx = context.WithValue(ctx, requestIDKey, id)    // valueCtx
ctx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
defer cancel()                                    // timerCtx -> cancelCtx -> valueCtx -> ...
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := http.DefaultClient.Do(req)
```

```text
timerCtx (300ms)
  └─ cancelCtx
       └─ valueCtx {requestID: "abc"}
            └─ 请求级 cancelCtx   ← 客户端断开从这里灌进来
```

`http.Client` 在 RoundTrip 里 `select` 这个 `Done`。到期、客户端断开、你手动 cancel，都会让 `Do` 返回。`Value` 沿链表能取到 `abc`，日志和 tracing 不用改 `Do` 的签名。

## 几个会写错的地方

**把 nil 当 context 传。** 标准库很多函数对 nil 会 panic。没有就 `context.Background()`，不要图省事。

**存在结构体里当「对象的一生」。** context 描述的是一次调用的生命周期，不是对象的生命周期。字段里存一个 ctx，过一段时间它取消了，对象还活着，后面的方法会莫名其妙失败。每次方法调用把 ctx 当参数传进去。

**在库函数里无条件 WithTimeout。** 超时策略属于调用方。库如果写死 3 秒，调用方那个 200ms 的 deadline 就被你盖掉了——更准确说，你无法放宽父的 deadline，但你会让「库内部又套一层」变得难以理解。库应该尊重传入的 ctx，把超时留给应用。

**忘记在循环里听 Done。** CPU 密集或大循环如果不定期看 `ctx.Err()`，取消信号要等到循环自己结束才有用。

```go
for i := range items {
    if err := ctx.Err(); err != nil {
        return err
    }
    process(items[i])
}
```

**用 Value 传必选参数。** 前面说过。签名里看得见的参数，才能被编译器和测试抓住。

**在 cancel 之后还用 ctx 启动新工作，却指望它继续跑。** 已经取消的 ctx，再 `WithCancel` 得到的孩子会立刻被取消。需要活过这次请求，用 `WithoutCancel` 或从 `Background` 重新开一枝，并且想清楚谁来收尾。

## 收住

`context` 的作用可以缩成一张表：

| 你想做的事 | 用什么 |
| --- | --- |
| 给调用链一个起点 | `Background` |
| 让下游停下来 | `WithCancel`，听 `Done` |
| 到期自动停 | `WithTimeout` / `WithDeadline` |
| 带请求 ID / trace | `WithValue`，key 用未导出类型 |
| 收尾工作不受请求取消影响 | `WithoutCancel`（想清楚再用不迟） |

实现上它是一棵树：`cancelCtx` 负责广播和级联，`timerCtx` 多一只闹钟，`valueCtx` 是向上查找的链表，`emptyCtx` 是永不取消的根。对你写业务代码而言，记住三件事就够往下传了：

1. ctx 放第一个参数，一直传到会阻塞的那一层
2. `WithCancel` / `WithTimeout` 的 `cancel` 用 `defer` 调掉
3. 阻塞处用 `select` 听 `ctx.Done()`，或调用同样听它的标准库 API

取消信号一旦成为调用约定，超时和泄漏就从「每个函数自己发明一个 quit channel」变成了语言里的一件小事。
