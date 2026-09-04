---
title: HTTP 装不成 RPC：从协议拆解到手写一个 Go RPC
date: 2026-09-04 16:20:00
categories: RPC
tags:
	- rpc
	- golang
	- net/rpc
	- protobuf
	- 源码阅读
---

本地函数调用只有两种结局：返回，或者崩。远程调用多出一个第三态——请求发出去了，你不知道对面执行过没有。RPC 要做的事，就是把这个第三态藏进一套约定，让业务代码写起来仍然像 `quo, rem, err := Divide(7, 3)`。

本文按五层往下拆：

1. **破冰**：RPC 是什么，为什么不能只用 HTTP/REST。
2. **进阶**：一次调用穿过哪些组件，黑盒里到底有什么。
3. **硬核**：序列化、分帧与多路复用、超时重试这三块怎么决定上限。
4. **源码**：Go 标准库 `net/rpc` 怎么把这套东西做进两百来行循环里。
5. **实战**：不依赖 `net/rpc`，从零写出一个能跑的极简 RPC。

## 一、破冰篇：什么是 RPC？

### 一次本地调用里，编译器替你做完了什么

```go
quo, rem, err := Divide(7, 3)
```

编译器和运行时已经约定好：参数怎么进寄存器或栈，返回值从哪取，调用双方共享同一份地址空间。指针能传，是因为对面和你看见的是同一块内存。类型对不上，编译期就拦下来了。

把 `Divide` 挪到另一台机器上，这些前提全部失效：

- 对面在哪？进程、机器、甚至一组机器里的哪一台。
- 参数怎么过去？内存布局不能直接搬，指针没有意义。
- 字节流怎么切开？TCP 只保证字节顺序，不保留消息边界。
- 回包是哪一次请求的？一条连接上往往同时飞着很多调用。
- 超时了怎么办？没回包可能是没收到、正在算、已经算完但回包丢了。三种情况对「能不能重试」的答案完全不同。

RPC（Remote Procedure Call）就是把寻址、编码、分帧、匹配、超时、错误语义打包掉，对外只留一个像本地函数的接口。1960 年代 RFC 707 就在谈这件事；今天的 gRPC、Thrift、Dubbo、`net/rpc` 换的是编码和传输，要解决的第三态没有变。

### RPC 是编程模型，不是某一种 HTTP 动词

一个容易混的点：RPC **不是**「不用 HTTP 的协议」。gRPC 跑在 HTTP/2 上；`net/rpc` 也能挂在 HTTP 的 `CONNECT` 上。RPC 说的是**调用形态**：

```text
结果, 错误 = 某个服务.某个过程(参数)
```

对照 REST：

```text
对某个资源做某个动词，得到一份表示
```

两者都能在网上传字节。差别在契约站在哪一边。

| | RPC | REST |
| --- | --- | --- |
| 核心抽象 | 过程 / 方法 | 资源 / 表示 |
| 契约 | IDL 或方法签名，强类型 | URL + 动词 + JSON schema，常常是弱约定 |
| 演化 | 加字段、加方法，向前兼容靠编号 | 加字段相对容易，URL 一改客户端全碎 |
| 流 | 双向流是一等公民（gRPC） | HTTP/1 基本是一次一响应 |
| 浏览器 | 不天然友好 | 人类和浏览器都能直接打 |
| 适用 | 服务与服务之间、内部 API | 对外的资源型 API、公开 Web |

所以「为什么不能只用 HTTP/REST」真正该问的是：**你的调用是不是一次过程调用**。内部微服务里 `Transfer(from, to, amount)` 不是对某个 `/accounts/42` 的 PATCH，硬塞进 REST 会得到一堆动词别扭的 URL，以及一份没有类型检查的 JSON。对外给第三方的资源目录，REST 仍然更合适。

HTTP/1.1 + JSON 当内部 RPC 用，还会撞上几堵具体的墙：

1. **没有多路复用**。一条连接同时只能等一个响应（pipeline 实际很少人开），高并发只能堆连接。HTTP/2 解决了这个，gRPC 因此选它。
2. **头和正文都偏胖**。每次调用重复传一堆 ASCII 头；JSON 数字、字段名都是文本。内部 QPS 上万时，CPU 和带宽都花在编解码上。
3. **没有标准的请求—应答匹配和截止时间**。你得自己在 body 里塞 `request_id`，自己用 `context` 取消，自己约定错误码。做完一遍，等于手写了一个残缺的 RPC 框架。
4. **契约是文档，不是编译器**。字段改名、类型从 `int` 变成 `string`，对面运行时才爆。IDL + 代码生成把这类错误提前到构建期。

一句话：HTTP 能当运输层；REST 是一种资源风格。RPC 要的是过程语义、强契约、以及把「执行过没有」这件事说清楚。运输层用 TCP 还是 HTTP/2，是下一层的选择，不是 RPC 本身。

### 远程调用多出来的第三态

本地 `Divide` 要么算出商，要么 panic。远程至少有四种你必须区分的失败：

- **没发出去**：连接断了、序列化失败。可以安全重试。
- **不知道到没到**：超时。重试可能让对面执行两次。
- **到了，业务拒绝**：余额不足。重试无意义，要把错误交给业务。
- **到了，框架崩了**：方法不存在、解码失败。通常不该重试同一份损坏请求。

RPC 框架如果把这四种错误都收成一个 `error`，调用方只能「超时就重试」，于是转账接口被打成双花。好的框架会强迫你看见这些区别——这是后文硬核篇要展开的调用语义。

## 二、进阶篇：剥开黑盒

一次 RPC 的最短路径：

```text
调用方                                              被调用方
业务代码
  |  像调本地函数
Client Stub  --序列化-->  Request{seq, method, argv}
  |                            |
  |                     定长头 + 变长体
传输 (TCP / HTTP/2)  ========================>  传输
                                   |
                          反序列化 --> Server Stub
                                   |  按方法名找到真正的函数
                                   业务代码
                                   |
                          Response{seq, err, reply}
  <================================
Client Stub  --反序列化-->  返回值交回业务
```

两侧的 stub 是整套幻觉的来源：调用方 stub 假装自己是那个函数；服务方 stub 假装自己是调用方。中间的编码和套接字，业务代码不该看见。

把黑盒再拆一层，生产级 RPC 通常是这些部件：

```text
┌─────────────────────────────────────────────────┐
│  IDL / 方法签名                                  │
│  生成 Client Stub + Server Skeleton              │
├─────────────────────────────────────────────────┤
│  拦截器：超时、鉴权、metrics、tracing             │
├───────────────┬─────────────────────────────────┤
│  序列化        │  协议（头：method、seq、timeout）│
├───────────────┴─────────────────────────────────┤
│  连接管理：池、多路复用、心跳、重连                │
├─────────────────────────────────────────────────┤
│  命名与发现：服务名 → 实例列表 → 负载均衡          │
└─────────────────────────────────────────────────┘
```

`net/rpc` 只实现了中间三层：反射当 stub、gob 当序列化、一条 TCP 连接上的请求应答。没有 IDL，没有发现，没有拦截器。gRPC 把每一层都做了，所以看起来重。

### 客户端真正在做什么

业务写 `client.Call("Arith.Divide", args, &reply)` 时，框架按顺序做：

1. 分配一个单调递增的 `seq`。
2. 把 `{seq, "Arith.Divide"}` 和参数分别编码进连接。
3. 在一张 `map[seq]*Call` 里登记这次调用，然后阻塞在 `call.Done` 上（异步则立刻返回这个 `Call`）。
4. 读循环读到响应头里的 `seq`，对上 map，把 reply 填回去，关掉 `Done`。

并发调用能共享一条连接，全靠这张 map。响应可以乱序到达——服务端每个请求一个 goroutine，先完成的先写回来。

### 服务端真正在做什么

1. **注册**：扫描对象的方法，按「导出、两参数、返回 error」过滤，建成 `服务名.方法名 → reflect.Method` 的路由表。生成代码派在这一步用 `.proto` 生成静态函数，省掉反射。
2. **读循环**：一条连接一个 goroutine 串行读。头读出来才知道 body 该解成什么类型。
3. **派发**：每个请求 `go` 出去执行，慢请求不堵住后面的请求。
4. **写回**：多 goroutine 写同一条连接必须加锁，否则两个响应的字节会交错。

读单线程、处理并发、写加锁——这是几乎所有「一条连接多个请求」的 RPC 服务端都会采用的形状。gRPC 用 HTTP/2 stream 替代了这把写锁的一部分工作，但「读、算、写」三段仍然在。

### 代理、骨架、拦截器

动态语言里常见「动态代理」：运行时生成一个实现了接口的对象，方法调用被拦截后发到网上。Go 没有完整的运行时代理，两条路：

- **反射 + 字符串方法名**：`net/rpc`。灵活，慢，没有编译期检查。
- **代码生成**：`protoc` 生成 `ArithClient` 接口和 `RegisterArithServer`。方法名、类型、字段编号全是生成出来的常量。

拦截器（middleware）插在 stub 和编码之间：先检查 deadline，再带上 trace id，再真正出站。它们不改变协议，只改变每次调用前后你能看见的东西。手写极简 RPC 时可以没有拦截器；一旦上生产，超时和 tracing 会从「业务自己写」变成「框架强制注入」。

## 三、硬核篇：三大核心技术

RPC 框架花里胡哨的功能都能卸，下面三件卸不掉。卸掉任何一件，你得到的就只是「在 socket 上 write 一段 JSON」。

### 1. 序列化：契约写在字节里

序列化要同时回答两个问题：内存对象怎么变成字节，以及**两边如何保证这是同一种对象**。

| 方案 | 契约 | 体积 / CPU | 跨语言 | 典型用户 |
| --- | --- | --- | --- | --- |
| JSON | 字段名，弱 | 大、慢 | 最好 | REST、调试 |
| Gob | Go 类型，自描述 | 中 | 仅 Go | `net/rpc` |
| Protobuf | `.proto` + 字段号 | 小、快 | 最好 | gRPC |
| Thrift / FlatBuffers | IDL | 小；FB 可零拷贝 | 好 | 特定栈 |

选择背后是兼容性策略：

- JSON 靠字段名。改名即断裂；加字段只要对方忽略未知键。
- Protobuf 靠字段号。改名无妨，**不要复用编号**。`optional` / `repeated` 的导线类型一旦公布就冻住。
- Gob 在流的开头发送类型定义，同一条连接上后续值很省。换一条连接就要再传一遍 schema，而且对面必须是 Go。

还有一件事常被忽略：**RPC 的参数必须是值，不能是指针图**。本地 `Divide` 可以收一个 `*Account` 再原地改余额；远程这么做，改的是反序列化出来的副本。要让对面改状态，只能把「改什么」做成参数，把「改完什么样」做成返回值。这是为什么 RPC 方法看起来总像纯函数，哪怕服务端内部全是副作用。

IDL 的价值不只是生成代码。它是一份**双方签字的契约**：字段编号、哪些能缺省、哪些会打破兼容。没有 IDL 的 JSON-RPC 把契约藏在 wiki 里，wiki 和代码的漂移是内部事故的常见来源。

### 2. 分帧、多路复用、连接模型

TCP 是字节流。你 `write` 了两次，对面可能 `read` 一次拿到两段，也可能反过来。RPC 必须自己定消息边界。常见三种：

**定长头 + 变长体**（后面实战篇用这个）：

```text
| 4 bytes length | length bytes payload |
```

实现十行，心跳、最大消息长度、半包都能处理。代价是自己定义 payload 里的 `seq` 和 `method`。

**自描述编码**：gob、部分 JSON 解码器知道一个值在哪结束。`net/rpc` 把分帧外包给 gob，所以它没有显式的 length 前缀。好处是简单；坏处是 gob 一崩，流就对不齐，只能断开连接。

**HTTP/2 帧**：gRPC 每个 RPC 一个 stream，stream id 就是多路复用的 seq。流控、优先级、头部压缩（HPACK/QPACK）都复用 HTTP/2。你不再手写 length prefix，但必须会调 HTTP/2 的机器。

多路复用解决的是「能不能一条连接上同时飞很多请求」。没有它，N 个并发调用就要 N 条连接，握手、fd、内存都会先爆。有了它，新问题是**队头阻塞**：

- HTTP/1.1 的队头阻塞在应用层：前一个响应没回来，后一个出不去。
- TCP 的队头阻塞在传输层：一个丢包，后面所有 stream 都等重传。这是 HTTP/3 / QUIC 要绕开的原因。
- 服务端如果**串行处理**请求，多路复用形同虚设。所以 `net/rpc` 读完立刻 `go service.call`。

连接模型还有心跳和重连。空闲连接会被 NAT 和负载均衡掐掉；心跳不是「保活玩玩」，是让调用方在发下一个请求之前就知道连接已经死了，而不是把超时预算浪费在一条死连接上。

### 3. 超时、重试、以及 exactly-once 的幻觉

超时不是「等不及了就报错」。它是把**截止时间**从调用方传到整个下游。Go 里这件事的载体是 `context.Context`：父调用 100ms 截止，子 RPC 必须带着同一个 ctx 走，否则叶子服务还在认真算，根已经给用户返回 504 了。`net/rpc` 的 `Call` **没有 context**，这是它被标成 frozen 的核心原因之一。

超时之后能不能重试，取决于你知不知道对面执行过：

```text
        请求丢失          执行中超时         响应丢失
       （可安全重试）    （重试=再执行一次）  （重试=再执行一次）
```

网络只能告诉你「我没等到」。框架如果提供自动重试，必须默认假设**执行可能已经发生**。于是：

- **只读请求**可以重试。
- **有副作用的请求**必须幂等：同一份 `Idempotency-Key` 执行两次，效果等于一次。
- **at-most-once**：超时就失败，不重试。丢单，不双花。
- **at-least-once**：超时就重试，直到成功或放弃。可能双花，业务自己去重。
- **exactly-once**：端到端几乎做不到。通常是「至少一次投递 + 服务端去重表」拼出来的效果。

错误也要分层，不能全是 `error`：

| 层 | 例子 | 调用方该做什么 |
| --- | --- | --- |
| 传输 | 连接 reset、decode 失败 | 换连接；必要时重试 |
| 框架 | 方法不存在、deadline exceeded | 修契约或加大超时；不要盲重试 |
| 业务 | `ErrInsufficientFund` | 交给业务；不要重试 |

gRPC 用 `codes.Unavailable` / `codes.InvalidArgument` / `codes.AlreadyExists` 把前两层标准化，业务错误放进 message。`net/rpc` 只在响应头里放一个字符串 `Error`，业务和框架挤在同一个 `error` 里——这是极简的代价。

这三块合在一起才是 RPC：字节怎么表示对象，字节怎么切成消息，消息失败时语义是什么。服务发现、负载均衡、熔断是规模上来之后的治理，不是 RPC 的最小定义。

## 四、源码篇：Go `net/rpc` 在干什么

标准库这个包文件头就写着 frozen，不接受新功能。它适合当教材，不适合当生产底座。读它，是因为它把上一节的三块做到了能运行的最小集，而且代码短。

约定接口：

```go
func (t *T) MethodName(arg T1, reply *T2) error
```

`T1`、`T2` 必须是可导出类型（或内建类型），`reply` 必须是指针，只能返回 `error`。每条限制都对应实现约束：gob 编不了未导出字段；框架要 `reflect.New` 出一块自己持有的 reply 内存；错误必须有唯一通道，好放进响应头。

### 注册：用反射建路由表

`Register` 扫方法，合法的放进 `service.method`。对外的名字默认是类型名，于是调用方写 `"Arith.Multiply"`。这一步没有 IDL，契约就是 Go 源码本身——对面必须是 Go，而且必须拼对字符串。拼错了要到运行时才知道。

### Codec：把协议从框架里拔出来

服务端真正依赖的不是 gob，而是：

```go
type ServerCodec interface {
    ReadRequestHeader(*Request) error
    ReadRequestBody(any) error
    WriteResponse(*Response, any) error
    Close() error
}
```

`Request` / `Response` 只有很少字段：

```go
type Request struct {
    ServiceMethod string
    Seq           uint64
}

type Response struct {
    ServiceMethod string
    Seq           uint64
    Error         string
}
```

`seq` 就是多路复用的钥匙。`Error` 非空时不再发送 body——业务失败和框架失败共用这一个字符串。

默认 `ServeConn` 把连接包成 `gobServerCodec`。换成 JSON 只要换 codec，这就是标准库 `net/rpc/jsonrpc` 的全部工作。分帧因此也交给编码器：gob 知道一个值在哪结束；头读成功但方法不存在时，必须再 `ReadRequestBody(nil)` 把 body 丢弃，否则流错位，后面所有请求全部解码失败。

### 服务循环：读串行、算并行、写加锁

`ServeCodec` 的骨架：

```text
for {
    读头 + 读 body
    if 头都读失败: break          // 流废了
    if 方法不存在: 回错误, continue
    go 执行方法, 加锁写响应
}
等所有 in-flight 写完
关连接
```

执行侧是 `service.call`：`reflect.Call`，然后 `sendResponse`。写路径抢 `sending *sync.Mutex`，保证两个 gob 值不会写穿。`net/rpc` 允许乱序响应，客户端必须靠 `seq` 对账，不能假设 FIFO。

### 客户端：pending map

`Client` 里最重要的是：

```go
type Client struct {
    codec ClientCodec
    seq   uint64
    pending map[uint64]*Call
    // mutex, closing, shutdown ...
}
```

`Go` 分配 `seq`、登记 `pending`、异步写请求；`Call` 只是 `Go` 完之后阻塞在 `call.Done`。另有一个读循环，收到响应就 `pending[seq]` 取出来填 reply。连接一断，读循环把 map 里所有 Call 都标成错误——这是「第三条状态」在标准库里的具体样子：所有 in-flight 调用同时失败，而且你不知道哪些已经在服务端跑完。

### 它故意没做的事

对照硬核篇，缺口一目了然：

- 没有 `context`，超时只能自己在外面套 `select`，而且取消传不进服务端。
- 没有拦截器，tracing / 鉴权要侵入业务。
- 没有流，没有推送。
- 错误只是字符串。
- 默认 gob，跨语言基本放弃。
- 没有服务发现。`Dial("tcp", addr)` 地址写死。

这些不是 bug，是 2010 年代「标准库给一个能用的最小 RPC」的边界。生产上该看 gRPC 或自研框架；读 `net/rpc` 是为了看见那条边界画在哪。

一个能跑的最小用法仍然值得记：

```go
// server
rpc.Register(new(Arith))
l, _ := net.Listen("tcp", ":1234")
rpc.Accept(l)

// client
c, _ := rpc.Dial("tcp", "127.0.0.1:1234")
var reply int
c.Call("Arith.Multiply", &Args{7, 8}, &reply)
```

下一节把同样的语义，不用 `net/rpc`，自己写一遍。写完你会对「框架到底替你藏了什么」不再有幻觉。

## 五、实战篇：从零构建一个极简 RPC

目标不是再造 gRPC，而是用最少的代码覆盖三大核心：

- 序列化：JSON（调试看得见）。
- 分帧：4 字节大端长度 + payload。
- 语义：`seq` 匹配；超时用 `context`；错误分传输错误和业务错误。

下面可以存成一个文件编译。为了篇幅，服务端用函数表而不是反射，方法集合写死——这反而更接近生成代码派：路由表在「编译期」就定了。

### 协议

```go
package minirpc

import (
	"context"
	"encoding/binary"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net"
	"sync"
	"sync/atomic"
	"time"
)

type Header struct {
	Seq    uint64 `json:"seq"`
	Method string `json:"method"`
	Error  string `json:"error,omitempty"`
}

type message struct {
	Header Header          `json:"header"`
	Body   json.RawMessage `json:"body"`
}

func writeMsg(w io.Writer, m message) error {
	payload, err := json.Marshal(m)
	if err != nil {
		return err
	}
	var lenBuf [4]byte
	binary.BigEndian.PutUint32(lenBuf[:], uint32(len(payload)))
	if _, err := w.Write(lenBuf[:]); err != nil {
		return err
	}
	_, err = w.Write(payload)
	return err
}

func readMsg(r io.Reader) (message, error) {
	var lenBuf [4]byte
	if _, err := io.ReadFull(r, lenBuf[:]); err != nil {
		return message{}, err
	}
	n := binary.BigEndian.Uint32(lenBuf[:])
	if n == 0 || n > 1<<20 {
		return message{}, fmt.Errorf("invalid frame length %d", n)
	}
	buf := make([]byte, n)
	if _, err := io.ReadFull(r, buf); err != nil {
		return message{}, err
	}
	var m message
	return m, json.Unmarshal(buf, &m)
}
```

`1<<20` 是防护：不让对面用一个 4GB 的 length 把内存打爆。生产框架都会有 max message size，这不是吹毛求疵。

### 服务端

```go
type MethodFunc func(ctx context.Context, in json.RawMessage) (any, error)

type Server struct {
	methods map[string]MethodFunc
}

func NewServer() *Server {
	return &Server{methods: make(map[string]MethodFunc)}
}

func (s *Server) Handle(name string, fn MethodFunc) {
	s.methods[name] = fn
}

func (s *Server) Accept(l net.Listener) error {
	for {
		conn, err := l.Accept()
		if err != nil {
			return err
		}
		go s.serveConn(conn)
	}
}

func (s *Server) serveConn(conn net.Conn) {
	defer conn.Close()
	var writeMu sync.Mutex
	for {
		req, err := readMsg(conn)
		if err != nil {
			return
		}
		go func(req message) {
			resp := s.dispatch(req)
			writeMu.Lock()
			_ = writeMsg(conn, resp)
			writeMu.Unlock()
		}(req)
	}
}

func (s *Server) dispatch(req message) message {
	fn, ok := s.methods[req.Header.Method]
	if !ok {
		return message{Header: Header{Seq: req.Header.Seq, Error: "method not found"}}
	}
	out, err := fn(context.Background(), req.Body)
	h := Header{Seq: req.Header.Seq}
	if err != nil {
		h.Error = err.Error()
		return message{Header: h}
	}
	body, _ := json.Marshal(out)
	return message{Header: h, Body: body}
}
```

对照 `net/rpc`：读循环在 `serveConn`，派发 `go` 出去，写用 `writeMu`。`Header.Seq` 原样返回。缺的是把 deadline 从请求头解出来放进 ctx——下一版加一个 `TimeoutMs` 字段即可。

### 客户端

```go
type Call struct {
	Seq    uint64
	Method string
	Reply  any
	Error  error
	Done   chan *Call
}

type Client struct {
	conn    net.Conn
	mu      sync.Mutex
	seq     uint64
	pending map[uint64]*Call
}

func Dial(addr string) (*Client, error) {
	conn, err := net.Dial("tcp", addr)
	if err != nil {
		return nil, err
	}
	c := &Client{conn: conn, pending: make(map[uint64]*Call)}
	go c.readLoop()
	return c, nil
}

func (c *Client) Call(ctx context.Context, method string, args, reply any) error {
	body, err := json.Marshal(args)
	if err != nil {
		return err
	}
	call := &Call{Method: method, Reply: reply, Done: make(chan *Call, 1)}

	c.mu.Lock()
	c.seq++
	call.Seq = c.seq
	c.pending[call.Seq] = call
	c.mu.Unlock()

	if err := writeMsg(c.conn, message{
		Header: Header{Seq: call.Seq, Method: method},
		Body:   body,
	}); err != nil {
		return err
	}

	select {
	case <-ctx.Done():
		c.mu.Lock()
		delete(c.pending, call.Seq)
		c.mu.Unlock()
		return ctx.Err()
	case <-call.Done:
		return call.Error
	}
}

func (c *Client) readLoop() {
	for {
		resp, err := readMsg(c.conn)
		if err != nil {
			c.failAll(err)
			return
		}
		c.mu.Lock()
		call, ok := c.pending[resp.Header.Seq]
		if ok {
			delete(c.pending, resp.Header.Seq)
		}
		c.mu.Unlock()
		if !ok {
			continue
		}
		if resp.Header.Error != "" {
			call.Error = errors.New(resp.Header.Error)
		} else if call.Reply != nil {
			call.Error = json.Unmarshal(resp.Body, call.Reply)
		}
		call.Done <- call
	}
}

func (c *Client) failAll(err error) {
	c.mu.Lock()
	defer c.mu.Unlock()
	for seq, call := range c.pending {
		call.Error = err
		call.Done <- call
		delete(c.pending, seq)
	}
}
```

`Call` 的 `select` 把硬核篇的超时接进来了：`ctx` 先结束就从 `pending` 摘掉。注意这仍然是 **at-most-once 的幻觉**：摘掉之后服务端可能还在跑，响应回来会被 `ok == false` 丢掉。如果你在超时后立刻重试，对面可能执行两次。极简实现把这个事实暴露出来，而不是用自动重试把它藏起来。

写请求这里为了短，没有单独的写锁。多个 `Call` 并发 `writeMsg` 可能把两个 length 前缀写穿。生产代码要么像 `net/rpc` 那样把发送丢进单个写 goroutine，要么对 `writeMsg` 加互斥。这是刻意留的缺口：看见它，才知道「能跑的 demo」和「能并发的框架」差在哪。

### 把它跑起来

```go
type Args struct{ A, B int }
type Quotient struct{ Quo, Rem int }

func main() {
	srv := NewServer()
	srv.Handle("Arith.Divide", func(_ context.Context, in json.RawMessage) (any, error) {
		var a Args
		if err := json.Unmarshal(in, &a); err != nil {
			return nil, err
		}
		if a.B == 0 {
			return nil, errors.New("divide by zero")
		}
		return Quotient{Quo: a.A / a.B, Rem: a.A % a.B}, nil
	})

	l, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		panic(err)
	}
	go srv.Accept(l)

	cli, err := Dial(l.Addr().String())
	if err != nil {
		panic(err)
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	var q Quotient
	if err := cli.Call(ctx, "Arith.Divide", Args{7, 3}, &q); err != nil {
		panic(err)
	}
	fmt.Println(q.Quo, q.Rem) // 2 1
}
```

到这里，五个方向合龙了：业务侧看起来仍是一次函数调用；底下是 JSON 契约、length-prefix 分帧、`seq` 多路复用、以及带 `context` 的超时。`net/rpc` 用 gob 和反射做了同一件事；gRPC 用 protobuf 和 HTTP/2 做了同一件事。换零件，不换问题。

### 从这一份再往前走

按出现顺序补，而不是一上来造轮子：

1. 给 `writeMsg` 加锁，或改成单写协程。
2. 请求头加 `timeout_ms`，服务端 `context.WithTimeout`。
3. 把 `Handle` 换成 codegen 或反射，去掉 `json.RawMessage`。
4. 业务错误用错误码，不要和 `method not found` 挤在同一个字符串。
5. 需要跨语言、流、拦截器时，停手，用 gRPC。

第五步不是认输。RPC 的价值在契约和语义，不在「我从零写了传输层」。教材写一遍，是为了读 gRPC 源码时知道 stream id 就是 `seq`，deadline 就是那个你差点忘了往下传的 `ctx`。

## 收束

RPC 不是「比 HTTP 更快的协议」，是「让远程过程看起来像本地调用」的一套约定。HTTP/REST 擅长资源和浏览器；内部过程调用需要的是类型化契约、消息边界、以及一张说得清的失败表。

架构上，stub、codec、连接、发现是分层的。硬核上，序列化、分帧与多路复用、超时重试决定正确性和上限。`net/rpc` 用反射和 gob 把前两层做到了最小；它缺的第三层（`context`、错误码、跨语言）正是后来者要补的。自己写一版 length-prefix JSON RPC，不是为了上线，是为了再也看不见黑盒。
