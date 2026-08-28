---
title: RPC 基本原理，以 Go net/rpc 源码为例
date: 2026-08-28 19:10:00
categories: RPC
tags:
	- rpc
	- golang
	- 源码阅读
---

RPC 的目标一句话就能说完：让调用远程机器上的一个函数，写起来和调用本地函数差不多。

难点全在「差不多」这三个字上。本地调用只会成功或 panic，远程调用多出一个第三态——你不知道对方到底执行了没有。本文先说清 RPC 要解决哪些问题，再把 Go 标准库 `net/rpc` 的实现从注册、收包、分发一路读到客户端的请求应答匹配。

## 从一次本地调用说起

本地调用长这样：

```go
quo, rem, err := Divide(7, 3)
```

编译器帮你做完了所有事：参数按调用约定压栈或进寄存器，跳到函数地址，返回值取回来。调用方和被调用方在同一个进程、同一份内存里，参数传的是指针也没问题。

把 `Divide` 挪到另一台机器上，这些前提全部失效：

- 对方在哪？要有地址，甚至要在一组地址里挑一个
- 参数怎么过去？内存里的对象要变成字节流，指针没有意义
- 字节流怎么切开？TCP 是流，不保留你的消息边界
- 响应回来了，是哪次请求的响应？
- 对方没回，是没收到、在处理、还是已经执行完但回包丢了？

RPC 框架就是把这一堆问题打包解决掉，对外只暴露一个像本地函数一样的接口。

## RPC 要解决的几件事

| 问题 | 通常的解法 |
| --- | --- |
| 寻址 | 服务名到地址的映射，注册中心 + 负载均衡 |
| 序列化 | 把参数和返回值在内存表示与字节流之间转换 |
| 分帧 | 定长头 + 变长体，或自描述格式 |
| 请求应答匹配 | 每个请求带唯一序号，响应回带同一序号 |
| 并发 | 一条连接上同时跑多个请求，允许乱序返回 |
| 错误语义 | 区分业务错误、框架错误、网络错误 |
| 超时与取消 | 调用方设定截止时间，到点放弃并释放资源 |

一次完整的调用链路：

```text
调用方                                          被调用方
业务代码
  |  像调本地函数一样调 stub
Client Stub  --- 序列化 --->  请求字节流
  |                                |
传输层(TCP)  ================>  传输层(TCP)
                                   |
                          反序列化 ---> Server Stub
                                   |  反射/生成代码找到真正的函数
                                   业务代码执行
                                   |
                          序列化响应 <--- 返回值
  <==============================
Client Stub  --- 反序列化 --->  返回值交回业务代码
```

两侧的 stub 是关键：调用方的 stub 假装自己是那个函数，被调用方的 stub 假装自己是调用方。中间怎么编码、怎么传输，业务代码不需要知道。

## Go net/rpc 的最小例子

Go 标准库自带一个 RPC 实现。先看它怎么用，再看它怎么写的。

需要注意，这个包已经冻结，源码开头就写着：

> The net/rpc package is frozen and is not accepting new features.

所以它适合用来理解原理，不适合直接上生产。后面会说清缺什么。

服务端：

```go
package main

import (
	"errors"
	"net"
	"net/rpc"
)

type Args struct{ A, B int }
type Quotient struct{ Quo, Rem int }

type Arith int

func (t *Arith) Multiply(args *Args, reply *int) error {
	*reply = args.A * args.B
	return nil
}

func (t *Arith) Divide(args *Args, quo *Quotient) error {
	if args.B == 0 {
		return errors.New("divide by zero")
	}
	quo.Quo = args.A / args.B
	quo.Rem = args.A % args.B
	return nil
}

func main() {
	rpc.Register(new(Arith))
	l, _ := net.Listen("tcp", ":1234")
	rpc.Accept(l) // 每来一条连接起一个 goroutine 服务
}
```

客户端：

```go
client, err := rpc.Dial("tcp", "127.0.0.1:1234")
if err != nil {
	log.Fatal(err)
}

// 同步调用
var reply int
if err := client.Call("Arith.Multiply", &Args{7, 8}, &reply); err != nil {
	log.Fatal(err)
}

// 异步调用
quo := new(Quotient)
call := <-client.Go("Arith.Divide", &Args{7, 3}, quo, nil).Done
```

业务代码里没有任何编解码和网络细节，只有 `"Arith.Multiply"` 这个方法名字符串。接下来看这层假象是怎么搭起来的。

## 服务端：注册阶段做了什么

`Register` 传进去一个对象，框架靠反射把它的方法挑出来当作可远程调用的方法。挑选规则写在 `suitableMethods` 里：

```go
func suitableMethods(typ reflect.Type, logErr bool) map[string]*methodType {
	methods := make(map[string]*methodType)
	for m := 0; m < typ.NumMethod(); m++ {
		method := typ.Method(m)
		mtype := method.Type
		mname := method.Name
		// 方法必须是导出的
		if !method.IsExported() {
			continue
		}
		// 必须是三个入参：receiver、*args、*reply
		if mtype.NumIn() != 3 {
			continue
		}
		argType := mtype.In(1)
		if !isExportedOrBuiltinType(argType) {
			continue
		}
		// 第二个参数必须是指针，因为要写返回值
		replyType := mtype.In(2)
		if replyType.Kind() != reflect.Pointer {
			continue
		}
		if !isExportedOrBuiltinType(replyType) {
			continue
		}
		// 必须只有一个返回值，且类型是 error
		if mtype.NumOut() != 1 {
			continue
		}
		if returnType := mtype.Out(0); returnType != typeOfError {
			continue
		}
		methods[mname] = &methodType{method: method, ArgType: argType, ReplyType: replyType}
	}
	return methods
}
```

于是有了那条著名的签名约束：

```go
func (t *T) MethodName(argType T1, replyType *T2) error
```

这些限制不是洁癖，每一条都对应一个实现需求：

- **参数恰好两个**：一个进、一个出，框架才知道该反序列化什么、该把什么序列化回去。返回值放在指针参数里而不是返回值列表里，是因为服务端要先 `reflect.New` 造出一个空对象交给你填，这样它自己就持有了这块内存
- **必须导出**：`encoding/gob` 只能编码导出字段，不导出的类型对面也拼不出来
- **只返回 error**：错误通道要统一。业务返回 error 时，框架把它转成字符串放进响应头，并且不再发送 reply

挑出来的结果存进 `service`，按服务名放进 `sync.Map`：

```go
type service struct {
	name   string                 // 服务名，默认取类型名
	rcvr   reflect.Value          // 方法的接收者
	typ    reflect.Type
	method map[string]*methodType // 方法名 -> 方法元信息
}
```

注册阶段本质上就是在建一张路由表：字符串 `"Arith.Multiply"` 到一个可以 `reflect.Call` 的函数。生成代码派（gRPC、brpc）在这一步用 IDL 提前生成 stub，省掉运行时反射，代价是多一道代码生成流程。

## 服务端：请求循环

`Accept` 对每条连接起一个 goroutine 跑 `ServeConn`，后者把连接包成一个 gob codec，再交给 `ServeCodec`：

```go
func (server *Server) ServeConn(conn io.ReadWriteCloser) {
	buf := bufio.NewWriter(conn)
	srv := &gobServerCodec{
		rwc:    conn,
		dec:    gob.NewDecoder(conn),
		enc:    gob.NewEncoder(buf),
		encBuf: buf,
	}
	server.ServeCodec(srv)
}
```

这里出现了整个包最重要的抽象——codec。协议细节全部收在这个接口后面：

```go
type ServerCodec interface {
	ReadRequestHeader(*Request) error
	ReadRequestBody(any) error
	WriteResponse(*Response, any) error
	Close() error
}
```

默认实现 `gobServerCodec` 就是四个 `gob.Encode` / `gob.Decode` 的包装。想换成 JSON，只要换一个 codec，框架其余部分一行不用改——标准库里的 `net/rpc/jsonrpc` 正是这么做的。

主循环本体很短：

```go
func (server *Server) ServeCodec(codec ServerCodec) {
	sending := new(sync.Mutex)
	wg := new(sync.WaitGroup)
	for {
		service, mtype, req, argv, replyv, keepReading, err := server.readRequest(codec)
		if err != nil {
			if !keepReading {
				break
			}
			// 头读到了就还能给对面一个错误响应，然后接着读下一个请求
			if req != nil {
				server.sendResponse(sending, req, invalidRequest, codec, err.Error())
				server.freeRequest(req)
			}
			continue
		}
		wg.Add(1)
		go service.call(server, sending, wg, mtype, req, argv, replyv, codec)
	}
	wg.Wait()
	codec.Close()
}
```

四个设计点都在这十几行里：

1. **读是单 goroutine 串行的**。一条连接上只有这个循环在读，不需要给读加锁
2. **业务处理另起 goroutine**。`go service.call(...)`，所以慢请求不会卡住后面的请求，代价是响应顺序和请求顺序可能不一致
3. **写用一把互斥锁串行化**。`sending` 传给每个 `service.call`，保证两个响应的字节不会交错
4. **优雅退出**。循环退出后 `wg.Wait()` 等所有在跑的请求把响应写完，再关 codec

### 消息怎么切开

`readRequest` 分两步：先读头，再按头里的方法名决定 body 该反序列化成什么类型。

```go
func (server *Server) readRequestHeader(codec ServerCodec) (svc *service, mtype *methodType, req *Request, keepReading bool, err error) {
	req = server.getRequest()
	err = codec.ReadRequestHeader(req)
	if err != nil {
		req = nil
		if err == io.EOF || err == io.ErrUnexpectedEOF {
			return
		}
		err = errors.New("rpc: server cannot decode request: " + err.Error())
		return
	}

	// 头读成功了，后面即使出错也还能继续处理下一个请求
	keepReading = true

	dot := strings.LastIndex(req.ServiceMethod, ".")
	if dot < 0 {
		err = errors.New("rpc: service/method request ill-formed: " + req.ServiceMethod)
		return
	}
	serviceName := req.ServiceMethod[:dot]
	methodName := req.ServiceMethod[dot+1:]

	svci, ok := server.serviceMap.Load(serviceName)
	if !ok {
		err = errors.New("rpc: can't find service " + req.ServiceMethod)
		return
	}
	svc = svci.(*service)
	mtype = svc.method[methodName]
	if mtype == nil {
		err = errors.New("rpc: can't find method " + req.ServiceMethod)
	}
	return
}
```

`keepReading` 这个返回值值得说一下。它区分了两类错误：

- 连头都没读出来（EOF、解码失败）：流已经不可信，只能断开
- 头读出来了，但服务或方法不存在：这一个请求废掉，回一个错误响应，连接继续用

能做到这一点，前提是能准确知道一个消息在哪结束。`net/rpc` 这里其实是把分帧问题外包给了 gob：gob 编码是自描述、自带长度的，`Decode` 一次就正好消费掉一个值。哪怕 body 的类型对不上，只要调一次 `ReadRequestBody(nil)` 把它读掉丢弃，流就还是对齐的——`readRequest` 里正是这么做的：

```go
service, mtype, req, keepReading, err = server.readRequestHeader(codec)
if err != nil {
	if !keepReading {
		return
	}
	// discard body
	codec.ReadRequestBody(nil)
	return
}
```

自己写协议就没这个便利，通常得设计成「固定长度的头里写上 body 长度」，收满长度再交给反序列化器。

拿到方法元信息后，按反射类型造出入参和出参：

```go
argIsValue := false
if mtype.ArgType.Kind() == reflect.Pointer {
	argv = reflect.New(mtype.ArgType.Elem())
} else {
	argv = reflect.New(mtype.ArgType)
	argIsValue = true
}
// 此时 argv 一定是指针，可以交给反序列化器填充
if err = codec.ReadRequestBody(argv.Interface()); err != nil {
	return
}
if argIsValue {
	argv = argv.Elem()
}

replyv = reflect.New(mtype.ReplyType.Elem())
```

不管你声明的入参是 `Args` 还是 `*Args`，反序列化时都需要一个指针，所以统一先 `reflect.New` 拿指针，值类型的最后再 `Elem()` 解引用回去。

### 真正的调用

`service.call` 就是一次反射调用加一次写响应：

```go
func (s *service) call(server *Server, sending *sync.Mutex, wg *sync.WaitGroup, mtype *methodType, req *Request, argv, replyv reflect.Value, codec ServerCodec) {
	if wg != nil {
		defer wg.Done()
	}
	mtype.Lock()
	mtype.numCalls++
	mtype.Unlock()
	function := mtype.method.Func
	returnValues := function.Call([]reflect.Value{s.rcvr, argv, replyv})
	// 方法的唯一返回值就是 error
	errInter := returnValues[0].Interface()
	errmsg := ""
	if errInter != nil {
		errmsg = errInter.(error).Error()
	}
	server.sendResponse(sending, req, replyv.Interface(), codec, errmsg)
	server.freeRequest(req)
}
```

`sendResponse` 里能看到业务错误的处理方式：一旦有错误信息，reply 就被换成一个空结构体，不再把可能填了一半的返回值发回去。

```go
resp.ServiceMethod = req.ServiceMethod
if errmsg != "" {
	resp.Error = errmsg
	reply = invalidRequest
}
resp.Seq = req.Seq
sending.Lock()
err := codec.WriteResponse(resp, reply)
sending.Unlock()
```

顺带一提，`Request` / `Response` 都是从空闲链表里取的：

```go
func (server *Server) getRequest() *Request {
	server.reqLock.Lock()
	req := server.freeReq
	if req == nil {
		req = new(Request)
	} else {
		server.freeReq = req.next
		*req = Request{}
	}
	server.reqLock.Unlock()
	return req
}
```

这是一个手写的对象池，用 `next` 指针串成链表复用头对象，减少高 QPS 下的分配。放到今天更自然的写法是 `sync.Pool`，但思路一样：RPC 框架里头对象是每请求都要创建的高频小对象。

## 客户端：序号与等待队列

服务端的模型是「一个请求一个 goroutine」，客户端反过来，是「一个连接一个读 goroutine，多个调用方共享」。核心状态就这么几个字段：

```go
type Client struct {
	codec ClientCodec

	reqMutex sync.Mutex // 保护下面这个 request，串行化发送
	request  Request

	mutex    sync.Mutex // 保护下面这些
	seq      uint64
	pending  map[uint64]*Call
	closing  bool // 本端调用了 Close
	shutdown bool // 对端告知不要再发了
}
```

`pending` 是整篇的关键：**序号到调用的映射**。有了它，一条连接上就能同时挂着很多个请求，响应回来时按序号找到对应的等待者，先回后回都不影响。

发送端：

```go
func (client *Client) send(call *Call) {
	client.reqMutex.Lock()
	defer client.reqMutex.Unlock()

	// 登记这次调用
	client.mutex.Lock()
	if client.shutdown || client.closing {
		client.mutex.Unlock()
		call.Error = ErrShutdown
		call.done()
		return
	}
	seq := client.seq
	client.seq++
	client.pending[seq] = call
	client.mutex.Unlock()

	// 编码并发送
	client.request.Seq = seq
	client.request.ServiceMethod = call.ServiceMethod
	err := client.codec.WriteRequest(&client.request, call.Args)
	if err != nil {
		client.mutex.Lock()
		call = client.pending[seq]
		delete(client.pending, seq)
		client.mutex.Unlock()
		if call != nil {
			call.Error = err
			call.done()
		}
	}
}
```

两把锁的分工要看清：

- `reqMutex` 锁住整个发送过程。因为 `WriteRequest` 要连着写头和体两个 gob 值，中间被另一个 goroutine 插进去，对面就解不出来了
- `mutex` 只保护 `seq` 和 `pending` 这几个字段，粒度尽量小

接收端是一个独立的 goroutine，在 `NewClientWithCodec` 里就启动了：

```go
func (client *Client) input() {
	var err error
	var response Response
	for err == nil {
		response = Response{}
		err = client.codec.ReadResponseHeader(&response)
		if err != nil {
			break
		}
		seq := response.Seq
		client.mutex.Lock()
		call := client.pending[seq]
		delete(client.pending, seq)
		client.mutex.Unlock()

		switch {
		case call == nil:
			// 没有对应的等待者，通常是 WriteRequest 半途失败已经清掉了。
			// body 还是要读掉，否则流会错位
			err = client.codec.ReadResponseBody(nil)
			if err != nil {
				err = errors.New("reading error body: " + err.Error())
			}
		case response.Error != "":
			// 服务端返回了错误，reply 是占位符，读掉丢弃
			call.Error = ServerError(response.Error)
			err = client.codec.ReadResponseBody(nil)
			if err != nil {
				err = errors.New("reading error body: " + err.Error())
			}
			call.done()
		default:
			err = client.codec.ReadResponseBody(call.Reply)
			if err != nil {
				call.Error = errors.New("reading body " + err.Error())
			}
			call.done()
		}
	}
	// 连接不可用了，把所有还在等的调用一次性唤醒并置错
	client.reqMutex.Lock()
	client.mutex.Lock()
	client.shutdown = true
	closing := client.closing
	if err == io.EOF {
		if closing {
			err = ErrShutdown
		} else {
			err = io.ErrUnexpectedEOF
		}
	}
	for _, call := range client.pending {
		call.Error = err
		call.done()
	}
	client.mutex.Unlock()
	client.reqMutex.Unlock()
}
```

循环退出后的收尾是容易被忽略但很重要的一段：连接断了，还挂在 `pending` 里的调用必须全部被唤醒并带上错误，否则调用方就永久卡在 channel 上了。任何自研 RPC 都得处理这件事。

`ReadResponseBody(nil)` 出现了两次，作用都是保持流对齐——不关心内容，但必须把这个消息消费掉。

### 同步调用只是异步调用的语法糖

```go
func (client *Client) Go(serviceMethod string, args any, reply any, done chan *Call) *Call {
	call := new(Call)
	call.ServiceMethod = serviceMethod
	call.Args = args
	call.Reply = reply
	if done == nil {
		done = make(chan *Call, 10) // 带缓冲
	} else {
		if cap(done) == 0 {
			log.Panic("rpc: done channel is unbuffered")
		}
	}
	call.Done = done
	client.send(call)
	return call
}

func (client *Client) Call(serviceMethod string, args any, reply any) error {
	call := <-client.Go(serviceMethod, args, reply, make(chan *Call, 1)).Done
	return call.Error
}
```

`Call` 就是 `Go` 之后立刻在 channel 上阻塞等一下。这也解释了为什么 `Done` 必须带缓冲：

```go
func (call *Call) done() {
	select {
	case call.Done <- call:
		// ok
	default:
		// 这里不能阻塞，缓冲不够是调用方的责任
	}
}
```

`done()` 是在 `input()` 这个唯一的读 goroutine 里被调用的。如果它在这里阻塞，整条连接上所有其他请求的响应都会被卡住。所以实现选择了 `select + default`：宁可丢掉这次通知（并打日志），也不能堵住读循环。用不带缓冲的 channel 会直接 panic，就是为了把这个错误暴露在开发阶段。

这是很典型的框架设计取舍：**IO 线程上的任何操作都不允许阻塞**。

## 三个可以直接搬走的设计

**codec 抽象**。协议编解码收在 `ServerCodec` / `ClientCodec` 后面，框架逻辑与线上格式解耦。同一套骨架配 gob 是 `net/rpc`，配 JSON 就是 `net/rpc/jsonrpc`。自研框架同样应该把序列化和协议做成可替换的。

**序号 + 等待队列**。请求带 `Seq`，响应回带同一个 `Seq`，客户端用 `map[uint64]*Call` 匹配。这是全双工、乱序返回、连接复用的基础。HTTP/1.1 缺的正是这个，所以只能靠开多条连接来并发；gRPC 用的 HTTP/2 stream id 本质上是同一个思路。

**读写职责分离**。读一个 goroutine 串行，写用互斥锁串行，业务处理各自起 goroutine。读写路径都不做重活，重活挪出去。

还有一个细节值得留意：`DialHTTP` 用 `CONNECT` 请求握手，服务端 `ServeHTTP` 里 `w.(http.Hijacker).Hijack()` 把连接抢出来，之后就完全走自己的协议了。这是在既有 HTTP 端口上复用 RPC 的经典做法。

## net/rpc 缺什么

理解原理它很够用，上生产则差得远。缺的东西恰好就是现代 RPC 框架的主要内容：

| 缺失 | 后果 |
| --- | --- |
| 没有 `context` | 不能超时、不能取消、不能传递链路信息。慢调用会一直占着 goroutine 和 `pending` 项 |
| 错误只剩字符串 | `ServerError` 是 string，业务错误码、错误类型全部丢失，调用方只能靠字符串匹配 |
| gob 绑定太紧 | 方法签名约束是按 gob 设计的，跨语言基本不可行 |
| 没有服务治理 | 服务发现、负载均衡、重试、熔断、限流、连接池，一样没有 |
| 没有流式 | 只有一问一答，推送和大数据流传输做不了 |
| 没有可观测性 | 除了一个 `numCalls` 计数和 debug 页面，没有指标、日志、链路追踪的接入点 |
| 包已冻结 | 不会再加特性 |

对照来看就清楚 gRPC、brpc、Kitex 这些框架在做什么：用 IDL 生成 stub 替代运行时反射，protobuf 替代 gob 拿到跨语言和向后兼容，HTTP/2 或自研协议提供多路复用和流式，再把服务发现、负载均衡、熔断、指标、追踪做成插件化的拦截器链。

## 自己写一个的话，最小清单

如果拿 `net/rpc` 当骨架自研，按优先级要补的是：

1. **分帧**：定长头写上 magic、版本、body 长度、压缩标志、序列化类型
2. **`context` 贯穿**：客户端 `Deadline` 到点就从 `pending` 里摘掉并返回超时；服务端把 ctx 透传到业务函数
3. **结构化错误**：错误码 + 消息 + 是否可重试，而不是一个 string
4. **连接池与多路复用**：一个下游地址维护若干长连接，每条连接上跑序号复用
5. **服务发现与负载均衡**：服务名解析出地址列表，按策略选一个，配合健康检查
6. **拦截器链**：把日志、指标、追踪、限流、熔断都挂在这一层，不要污染业务代码
7. **元数据透传**：trace id、染色标记这类跨服务传递的信息，需要协议头留位置

## 小结

RPC 的实现原理并不神秘，`net/rpc` 用一千多行就把骨架写完了：注册阶段用反射建一张 `"服务.方法"` 到函数的路由表；服务端单 goroutine 读、按序号回、业务处理另起 goroutine；客户端用 `seq` 和 `pending` 把请求与响应对上，同步调用只是等一个 channel。

真正把框架撑大的是它没做的那部分——超时取消、结构化错误、连接管理、服务治理、可观测性。读它的价值在于，把「RPC 到底做了什么」这件事从概念变成一条能跟到底的调用路径；之后再看 gRPC 或 brpc 的复杂度，就知道每一层是在解决哪个具体问题。
