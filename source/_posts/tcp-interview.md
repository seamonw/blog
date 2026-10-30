---
title: TCP/IP 面试全解：握手挥手状态机、丢包重传、滑动窗口与拥塞控制
date: 2026-10-30 15:20:00
categories: 网络
tags:
	- tcp
	- 面试
	- 状态机
	- 滑动窗口
	- 拥塞控制
---

客户端 `connect()` 已经返回成功，第一笔请求发出去却石沉大海；或者线上连接数不高，客户端却频繁报 `Cannot assign requested address`。前者是三次握手的最后一个 `ACK` 被打满的全连接队列静默丢弃，客户端单方面进入了 `ESTABLISHED`；后者是短连接由客户端主动关闭，`TIME_WAIT` 把本地临时端口耗尽。

本文把 TCP/IP 后台面试从「头部与分片」「三次握手与双队列」「四次挥手与 `TIME_WAIT`」「重传、滑动窗口与流量控制」「拥塞控制」到「断电/崩溃/旧包等异常边界」收敛成一篇。内核参数与常数以 Linux `include/net/tcp.h` 为准。

## 一、TCP 头部、四元组与为什么需要 MSS

一条 TCP 连接由**四元组**（源 IP、源端口、目的 IP、目的端口）唯一确定。TCP 与 UDP 在内核中属于两个独立的协议模块，端口号空间互不干扰，因此**同一个机器上的 TCP 和 UDP 可以同时绑定同一个端口号**。

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |       |C|E|U|A|P|R|S|F|                               |
| Offset| Rsrvd |W|C|R|C|S|S|Y|I|            Window             |
|       |       |R|E|G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (0 ~ 40 Bytes)                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

TCP 头部固定 20 字节，`Data Offset`（4 位，单位为 4 字节）决定了带选项时头部最长 **60 字节**。

**既然 IP 层会自动分片，为什么 TCP 层还需要 `MSS`（最大报文段长度）？**
IP 层没有超时重传机制。如果把一个 10 KB 的 TCP 报文直接交给 IP 层切成 8 个 IP 分片，途中只要**丢掉其中 1 个 IP 分片**，接收方 IP 层就无法重组上交，发送方 TCP 超时后必须**重传整个 10 KB 报文**。在三次握手中协商 `MSS`（以太网 `MTU 1500 - 20 IP头 - 20 TCP头 = 1460` 字节），由 TCP 层按 `MSS` 切段，丢哪一段就只重传那一个 `MSS`。

---

## 二、三次握手：状态机、各步丢包与内核双队列

### 1. 为什么是三次握手，不是两次或四次？

1. **防止历史重复连接初始化造成服务端资源浪费（核心原因）**：网络拥塞时，客户端旧的 `SYN(seq=90)` 滞留在网中，超时后重发了新的 `SYN(seq=100)`。若旧 `SYN` 先到达服务端，服务端回 `SYN-ACK(ack=91)`；客户端收到后发现自己当前期望的是 `ack=101`，识别出这是历史连接，直接发 `RST` 中止，服务端不会建立连接。**若是两次握手，服务端发完 `SYN-ACK` 就直接进入 `ESTABLISHED` 并分配资源，挡不住旧 `SYN`。**
2. **同步双方初始序列号（ISN）**：一来一回只能确认一方的序列号被收到；服务端的 `ACK` 和 `SYN` 没有业务数据要等，可以合并成一个 `SYN-ACK` 包，不需要四次。

```text
客户端                                    服务端（listen 后处于 LISTEN）
CLOSED
  |  connect()：发 SYN(seq=x)
  v
SYN_SENT  ----------------------------->  收到 SYN：入半连接队列(SYN队列)
                                          回 SYN-ACK(seq=y, ack=x+1)
                                          状态：SYN_RECV
  |  收到 SYN-ACK：发 ACK(ack=y+1)
  |  （第 3 次握手可携带数据）
  v
ESTABLISHED --------------------------->  收到 ACK：从 SYN 队列移入全连接队列(accept队列)
                                          状态：ESTABLISHED
                                          应用调 accept() 从全连接队列取走已建立的 fd
```

- **为什么每次建连的初始序列号 `ISN` 都不同？** 一是防止防伪造盲打攻击，二是防止前一条相同四元组连接残留在网络中的历史报文被新连接误收。Linux 用每 4 微秒加 1 的时钟加上四元组哈希生成 `ISN`。在万兆网下 32 位序列号几秒就会回绕，需配合 TCP `Timestamps` 选项与 **PAWS（防序列号回绕）** 丢弃时间戳倒退的旧包。
- **哪次握手能带数据？** 前两次不能（防止攻击者在未确认身份前塞大包消耗服务端内存），第三次握手时客户端已处于 `ESTABLISHED`，可以携带数据（开启 `TCP_FASTOPEN` 且握有 Cookie 时，第一次 `SYN` 才允许带数据）。

### 2. 第 1 / 2 / 3 次握手丢包会发生什么？

| 丢失报文 | 客户端行为与状态 | 服务端行为与状态 | 控制参数（Linux 默认值） |
| --- | --- | --- | --- |
| **第 1 次 `SYN` 丢** | 停在 `SYN_SENT`，超时重传相同 seq 的 `SYN`（退避 1s, 2s, 4s...） | 无感知，保持 `LISTEN` | `tcp_syn_retries`（默认 `6` 次，约 127s 超时） |
| **第 2 次 `SYN-ACK` 丢** | 以为 `SYN` 丢了，按 `tcp_syn_retries` 重传 `SYN` | 停在 `SYN_RECV`，收不到第 3 次 `ACK`，超时重传 `SYN-ACK` | 客户端 `tcp_syn_retries=6`；服务端 `tcp_synack_retries=5` |
| **第 3 次 `ACK` 丢** | **已进入 `ESTABLISHED`**。若未发数据，收到重传的 `SYN-ACK` 会重回 `ACK`；若紧接着发了数据包（自带 `ACK` 位），服务端收到直接建连 | 停在 `SYN_RECV`，按 `tcp_synack_retries` 重传 `SYN-ACK`，5 次全败则销毁半连接 | `tcp_synack_retries`（默认 `5` 次，约 31s） |

### 3. 半连接队列、全连接队列与 SYN 攻击

- **没有 `listen` 能建连吗？** 不能，服务端内核直接回 `RST`。
- **没有调用 `accept()` 能完成三次握手吗？** **能。** 三次握手完全由内核协议栈完成，建好的连接放在全连接队列里；`accept()` 只是把连接从全连接队列取到用户态。

```text
SYN 到达 ---> [ 半连接队列 (SYN Queue) ] ---> 收到第3次 ACK ---> [ 全连接队列 (Accept Queue) ] ---> accept()
              上限: tcp_max_syn_backlog                        上限: min(backlog, somaxconn)
              满时: tcp_syncookies=1 免队列计算 Cookie          满时: tcp_abort_on_overflow=0 静默丢弃 ACK
                                                                     tcp_abort_on_overflow=1 直接回 RST
```

- **半连接队列满（SYN Flood 攻击）**：调大 `netdev_max_backlog` 与 `tcp_max_syn_backlog`，开启 **`tcp_syncookies = 1`**（队列满时把连接信息哈希进 `SYN-ACK` 的序列号中，不占队列槽位，收到合法的第 3 次 `ACK` 直接进全连接队列），并调小 `tcp_synack_retries`。
- **全连接队列满（线上偶发超时元凶）**：当服务端处理慢、来不及 `accept` 时，全连接队列打满。默认 **`tcp_abort_on_overflow = 0`** 会**直接丢弃客户端发来的第 3 次握手 `ACK` 和后续数据包**，服务端留在 `SYN_RECV` 重传 `SYN-ACK`（赌重传期间应用腾出了队列空位）；若设为 `1` 则直接回 `RST`（客户端报 `Connection reset by peer`）。
- **如何一眼看出全连接队列打满？** 看 `ss -lnt`。在 `LISTEN` 状态下，**`Recv-Q` 是当前全连接队列排队数，`Send-Q` 是全连接队列最大上限**（而在非 `LISTEN` 的已建立连接上，`Recv-Q` / `Send-Q` 才是接收/发送缓冲区字节数）。

---

## 三、四次挥手：状态机、各步丢包与 TIME_WAIT

### 1. 为什么挥手要四次？什么时候能合并成三次？

TCP 是全双工的。主动方调 `close()` 发 `FIN`，只代表「我不再发数据了」，但被动方可能还有剩余数据没发完。所以被动方先回 `ACK`（进入 `CLOSE_WAIT`），等自己数据发完、应用调 `close()` 时再发 `FIN`（进入 `LAST_ACK`）。

**四次挥手合并成三次的条件**：被动关闭方收到 `FIN` 时**恰好没有数据要发送**，并且**开启了 TCP 延迟确认（默认开启）**，此时应用层在延迟确认超时（40ms）内迅速调用了 `close()`，内核就会把第 2 次挥手的 `ACK` 和第 3 次挥手的 `FIN` 合并成一个 `FIN+ACK` 包发出。

```text
主动关闭方                                 被动关闭方
ESTABLISHED                               ESTABLISHED
  |  close() / shutdown(SHUT_WR)：发 FIN
  v
FIN_WAIT_1 -----------------------------> 收到 FIN，回 ACK
                                          状态：CLOSE_WAIT（等本端应用调 close）
  |  收到 ACK
  v
FIN_WAIT_2                                发送剩余数据...
  |                                       应用调 close()：发 FIN
  |        <----------------------------- 状态：LAST_ACK
  |  收到 FIN，回 ACK
  v
TIME_WAIT  -----------------------------> 收到 ACK：CLOSED
  |  等待 2MSL（Linux 固定 60 秒）
  v
CLOSED

同时关闭路径：双方同时发 FIN 进 FIN_WAIT_1 -> 收到对端 FIN 回 ACK 进 CLOSING -> 收到 ACK 进 TIME_WAIT
```

### 2. 第 1 / 2 / 3 / 4 次挥手丢包会发生什么？

| 丢失报文 | 主动关闭方行为与状态 | 被动关闭方行为与状态 | 控制参数（Linux 默认值） |
| --- | --- | --- | --- |
| **第 1 次 `FIN` 丢** | 停在 `FIN_WAIT_1`，超时重传 `FIN`，超限后直接 `CLOSED` | 无感知，保持 `ESTABLISHED` | `tcp_orphan_retries`（默认 `0`，内核按 `8` 次算） |
| **第 2 次 `ACK` 丢** | 停在 `FIN_WAIT_1`（`ACK` 不会重传），超时重传第 1 次的 `FIN` | 已进 `CLOSE_WAIT`，收到重复 `FIN` 后重回 `ACK` | 同上 |
| **第 3 次 `FIN` 丢** | 停在 `FIN_WAIT_2`。若由 `close()` 进入，等 `tcp_fin_timeout` 到期直接关闭；若由 `shutdown(SHUT_WR)` 进入，一直等 | 停在 `LAST_ACK`，收不到第 4 次 `ACK`，超时重传 `FIN` | 主动方 `tcp_fin_timeout=60s`；被动方 `tcp_orphan_retries` |
| **第 4 次 `ACK` 丢** | 已进 `TIME_WAIT`。收到被动方重传的 `FIN` 后，重回 `ACK` **并将 2MSL 计时器重置为 60s** | 停在 `LAST_ACK`，超时重传第 3 次的 `FIN`，超限后关闭 | `TCP_TIMEWAIT_LEN`（60s）与 `tcp_orphan_retries` |

### 3. TIME_WAIT 的 2MSL、端口复用与 CLOSE_WAIT

**为什么需要 `TIME_WAIT` 且固定等待 `2MSL`（Linux 中 `MSL = 30s`，`2MSL = 60s`）？**
1. **让历史连接的所有滞留报文在网络中自然消亡**：防止旧数据包被后面复用相同四元组的新连接当成有效数据接收。
2. **确保被动关闭方能正常关闭**：如果第 4 次挥手的 `ACK` 丢了，最大 `1 MSL` 内该 `ACK` 消失，被动方超时重传的 `FIN` 在接下来的 `1 MSL` 内到达主动方，一来一回正好 `2MSL`。

**`TIME_WAIT` 过多 vs `CLOSE_WAIT` 过多：**
- **服务端大量 `CLOSE_WAIT`**：100% 是本端代码 bug——对端已经关了连接，本端没有调 `close()`（fd 泄漏，或关连接前卡在数据库/锁上）。调任何内核参数都没用。
- **客户端大量 `TIME_WAIT`**：占满 `ip_local_port_range` 导致无法向同一目标发起新连接。可开启 `net.ipv4.tcp_tw_reuse = 1`（要求开启 `tcp_timestamps`，**仅对客户端向外 `connect()` 生效**，复用处于 `TIME_WAIT` 超过 1 秒的连接）。注意：**`tcp_tw_recycle` 在 Linux 4.12 已被彻底删除**，因为在 NAT 环境下多台机器时间戳不单调，会导致服务端把正常的新 `SYN` 当成旧包丢弃。
- **服务端大量 `TIME_WAIT`**：说明服务端在主动关连接（短连接未开 HTTP Keep-Alive，或 Nginx 到后端未用连接池）。`tcp_tw_reuse` 对监听收进来的连接无效，只能改长连接或让客户端主动关。
- **Socket 选项辨析**：
  - `SO_REUSEADDR`：允许服务端重启时，哪怕旧连接还在 `TIME_WAIT` 占着端口，也能立刻 `bind()` 成功（服务端必开）。
  - `SO_REUSEPORT`：允许多个进程/线程同时 `bind()` 同一个 IP+端口，内核按四元组哈希把连接均匀分发到不同 `accept` 队列，消除单 listen 锁争用。
  - `SO_LINGER`：设 `l_onoff=1, l_linger=0` 时，`close()` 不发 `FIN` 而直接发 `RST`，跳过 `TIME_WAIT`，但发送缓冲区里未发完的数据直接丢弃。

---

## 四、可靠传输与流量控制：SACK、滑动窗口与糊涂窗口

### 1. 超时重传、快速重传、SACK 与 D-SACK

- **超时重传**：`RTO` 动态跟踪略大于 `RTT`（Linux 下限 `TCP_RTO_MIN = 200ms`），每次重传 `RTO` 翻倍。代价是等待周期太长。
- **快速重传**：接收方每收到一个乱序包，就立即重发「当前期望序号」的 `ACK`。发送方收到 **3 个相同的重复 `ACK`**（连同最初那 1 个共 4 个），不等超时立即重传。**缺陷**：不知道是只丢了 1 个包，还是后面连续丢了几个包。
- **SACK（选择性确认）**：在 TCP 头部 `Options` 中带上「已收到的乱序数据区间」（最多 4 段），发送方据此**只重传真正缺失的空洞**（`tcp_sack = 1` 默认开启）。
- **D-SACK（Duplicate SACK）**：用第一个 SACK 块告诉发送方「哪些数据我收到了重复份」。三个实战作用：
  1. 让发送方分清是**发出去的数据包丢了**，还是**回来的 `ACK` 丢了**；
  2. 判断是否因为网络短暂延迟导致 `RTO` 过早触发了虚假重传；
  3. 判断网络设备是否异常复制了报文。

### 2. 滑动窗口指针与 Window Scale

发送缓冲区在内核里被三个指针切成四段：

```text
  #1 已发送并已确认    #2 已发送但未确认      #3 未发送但可立即发送     #4 超出窗口不可发送
-------------------|---------------------|------------------------|-------------------
                   ^                     ^                        ^
                SND.UNA               SND.NXT           SND.UNA + SND.WND
                   |<------------- 发送窗口 SND.WND ------------->|
                                         |<-- 可用窗口 Usable --->|
```

- **可用窗口公式**：`Usable = SND.WND - (SND.NXT - SND.UNA)`。
- **Window Scale（窗口缩放）**：TCP 头部的 `Window` 字段只有 16 位，最大只能表示 **64 KiB**。在三次握手的 `SYN` 中协商 `Window Scale`（左移最多 14 位），可将窗口扩到 **1 GiB**。在跨国高延迟链路上（例如带宽 1 Gbps、`RTT = 100ms`，带宽时延积 `BDP = 12.5 MB`），如果防火墙剥离了 `Window Scale` 选项，单连接吞吐上限会被死死卡在 `64 KiB / 0.1s ≈ 640 KB/s`。

### 3. 零窗口探测与糊涂窗口综合症

- **窗口关闭与零窗口探测（ZWP）**：接收方处理不过来时通告 `rwnd = 0`。如果之后接收方腾出空位发出的非零窗口 `ACK` 在网中丢了（`ACK` 不重传），双方就会死锁。因此发送方在收到 `rwnd = 0` 后会启动**持续计时器（Persist Timer）**，周期性发送 1 字节的零窗口探测包，逼接收方回包通告最新窗口。
- **糊涂窗口综合症（Silly Window Syndrome）**：接收方刚腾出几个字节就通告小窗口，发送方立刻发几个字节的小包，40 字节的 TCP+IP 头远比有效数据还大。
  - **接收方解法**：可用空间小于 `min(MSS, 接收缓冲/2)` 时，一律对外通告窗口为 `0`。
  - **发送方解法（Nagle 算法）**：只有满足「没有在途未确认数据（`SND.UNA == SND.NXT`）」**或**「待发数据攒够 `1 MSS`」时才发包，否则在缓冲区里等着。
- **Nagle 撞上延迟 ACK（经典 40ms 毛刺）**：延迟确认规定，收到数据后若没有响应数据可捎带，就等 `TCP_DELACK_MIN`（Linux 默认 **40ms**）再回纯 `ACK`。若客户端开着 Nagle，把一个请求分两次 `write(header)` + `write(body)` 发送，而服务端必须收齐 `body` 才回包：第一次 `header` 发出后，客户端因 Nagle 卡住 `body` 等 `header` 的 `ACK`，服务端又因延迟确认等 40ms 才回 `header` 的 `ACK`，白白卡死 40ms。**解法**：开启 `TCP_NODELAY`（Go 标准库默认已开）或用 `writev` 一次写完。

---

## 五、拥塞控制：慢启动、快恢复与 CUBIC / BBR

流量控制看 `rwnd`（防撑爆接收方缓冲），拥塞控制看 `cwnd`（防打崩中间网络），实际发送窗口 `swnd = min(cwnd, rwnd)`。

```text
cwnd (单位: MSS)
  ^
  |            / 拥塞避免: 每 RTT +1        收到 3 个重复 ACK (快重传 + 快恢复):
  |           /                            1. ssthresh = cwnd / 2
ssthresh ----+            *\               2. cwnd = ssthresh + 3 (3个乱序包已到接收方)
  |         /             | \              3. 每多收1个重复 ACK, cwnd += 1
  |        / 慢启动:      |  \             4. 收到新数据 ACK, cwnd = ssthresh, 进拥塞避免
  |       /  每新 ACK +1  |   +---
10|------+   (每RTT翻倍)  |                RTO 超时重传:
  |                       |                ssthresh = cwnd / 2, cwnd 直接打回 1!
  +---------------------------------> 时间
```

下面这段可运行的 Go 代码把滑动窗口可用量与 Reno 四阶段（含快恢复 `+3` 膨胀细节）完整串起来：

```go
package main

import "fmt"

type tcpConn struct {
	sndUna   int // 已发送未确认的起始序号
	sndNxt   int // 下一个待发送序号
	rwnd     int // 对端通告的接收窗口
	cwnd     int // 拥塞窗口（均以 MSS 为单位）
	ssthresh int // 慢启动门限
}

func (c *tcpConn) usableWindow() int {
	swnd := min(c.cwnd, c.rwnd)
	inFlight := c.sndNxt - c.sndUna
	return max(swnd-inFlight, 0)
}

func (c *tcpConn) onRTT() {
	if c.cwnd < c.ssthresh {
		c.cwnd *= 2 // 慢启动：每 RTT 翻倍
		return
	}
	c.cwnd++ // 拥塞避免：每 RTT +1
}

func (c *tcpConn) onThreeDupACK() {
	c.ssthresh = max(c.cwnd/2, 2)
	c.cwnd = c.ssthresh + 3 // 快恢复：+3 代表触发重复 ACK 的 3 个段已离开网络
}

func (c *tcpConn) onDupACKInFastRecovery() {
	c.cwnd++ // 快恢复期间每再收一个重复 ACK，窗口 +1
}

func (c *tcpConn) onNewACKAfterRecovery() {
	c.cwnd = c.ssthresh // 收到确认新数据的 ACK，回落到 ssthresh 进入拥塞避免
}

func (c *tcpConn) onRTO() {
	c.ssthresh = max(c.cwnd/2, 2)
	c.cwnd = 1 // 超时重传：cwnd 直接打回 1
}

func main() {
	c := tcpConn{sndUna: 100, sndNxt: 106, rwnd: 32, cwnd: 10, ssthresh: 16}
	fmt.Println("usable", c.usableWindow()) // 4

	c.onRTT()
	fmt.Println("slow start cwnd", c.cwnd) // 20
	c.onRTT()
	fmt.Println("avoidance cwnd", c.cwnd) // 21

	c.onThreeDupACK()
	fmt.Println("fast recovery start", c.cwnd, c.ssthresh) // 13 10
	c.onDupACKInFastRecovery()
	fmt.Println("fast recovery inflate", c.cwnd) // 14
	c.onNewACKAfterRecovery()
	fmt.Println("exit recovery", c.cwnd) // 10

	c.onRTO()
	fmt.Println("after RTO", c.cwnd, c.ssthresh) // 1 5
}
```

**CUBIC vs BBR 怎么选？**
- **CUBIC（Linux 默认）**：用三次函数曲线增长，但依然**把丢包等同于拥塞**。在跨洋或弱网等高延迟、偶发随机丢包（如 1%）的链路上，每次丢包都砍窗口，吞吐会暴跌。
- **BBR**：不把单次丢包当降窗信号，而是持续测量**最大瓶颈带宽（BtlBw）**和**最小往返延迟（RTprop）**，将发包速率与在途数据量锚定在 `BDP = BtlBw × RTprop`，在有轻微随机丢包的链路上吞吐优势巨大。应用层在 UDP 上自建重传与窗口的取舍，见 [KCP 源码阅读：可靠 UDP 的 ARQ 与按序交付](/blog/2026/09/09/kcp-source/)。

---

## 六、异常边界与面试高频/深度追问

### 1. 进程崩溃 vs 拔掉网线（或机器断电）

这是面试最常追问的对比，核心判据是**对端内核还活没活着、能不能替它发包**：

| 场景 | 发生什么 | 最终耗时与控制参数 |
| --- | --- | --- |
| **对端进程崩溃（`kill -9`）** | 对端内核还活着，自动回收该进程的所有 socket fd 并**主动发出 `FIN`**，完成正常四次挥手 | 立即感知 |
| **拔网线 / 断电，之后有数据发送** | 发送方收不到 `ACK`，持续超时重传数据包。若重传期间插回网线，连接直接恢复；若一直不通，重传超限后报错断开 | `tcp_retries2`（默认 `15` 次，指数退避累计约 **13 ~ 30 分钟**） |
| **拔网线 / 断电，之后无数据发送** | 未开 `SO_KEEPALIVE` 时连接**永远处于 `ESTABLISHED`**；开启后等空闲 2 小时才开始发探测包，9 次无响应才断开 | `tcp_keepalive_time=7200s` + `intvl=75s` × `probes=9` ≈ **2 小时 11 分 15 秒** |
| **对端断电重启完成，本端再发数据** | 对端内核已丢失旧连接上下文，收到数据包后**直接回 `RST`**，本端报 `Connection reset by peer` | 收到包立刻断开 |

生产环境不能指望 2 小时的 TCP Keepalive，必须在应用层自己做心跳与空闲超时，定时器实现见 [时间轮算法：原理、层次化实现和一组 Go Timer 接口](/blog/2026/08/28/timing-wheel/)。

### 2. 状态冲突：已建立连接或 TIME_WAIT 收到 SYN、乱序 FIN

- **处于 `ESTABLISHED` 的连接收到 `SYN` 会怎样？**
  比如客户端断电重启后，碰巧用相同的源端口向服务端发起新连接 `SYN`，而服务端还留着旧的 `ESTABLISHED` 连接。服务端收到序列号不对的 `SYN` 后，不会直接建连，而是回一个携带当前期望确认号的 **Challenge ACK**；客户端收到后发现确认号和自己刚发的 `SYN` 对不上，立刻回 **`RST`** 把服务端的旧连接拆掉，随后重发 `SYN` 正常建连。
- **处于 `TIME_WAIT` 的连接收到 `SYN` 会怎样？**
  若新 `SYN` 的序列号大于旧连接结束时的最大序列号（开启 `Timestamps` 时时间戳也严格递增），内核判定这是合法的新连接，直接跳过 2MSL 复用四元组建连；若序列号/时间戳更旧，则视为历史重复 `SYN`，回一个旧连接的最后 `ACK`（对端随后回 `RST`）。
- **四次挥手中收到乱序的 `FIN` 包会怎样？**
  在 `FIN_WAIT_2` 收到对端的 `FIN`，但 `FIN` 的序列号大于当前期望接收序号 `RCV.NXT`（说明 `FIN` 前面的数据包还没到）。此时内核**不会立刻回 `FIN` 的 `ACK`，也不会进 `TIME_WAIT`**，而是把 `FIN` 放进乱序队列（`out_of_order_queue`），等前面的空洞数据包全部补齐后，才处理 `FIN` 进入 `TIME_WAIT`。

### 3. 高频追问速查表

| 面试问题 | 一句话到位回答 |
| --- | --- |
| 为什么 TCP 粘包，UDP 不会？ | TCP 是面向字节流的，无消息边界；UDP 面向报文，每次 `recvfrom` 对应一个完整数据报。分帧做法见 [RPC 原理、Go net/rpc 与一份极简实现](/blog/2026/09/04/rpc-from-scratch/) |
| 用了 TCP 数据就一定不丢吗？ | 不是。TCP 只保证传输层到对端内核缓冲区的可靠；`write` 前崩溃、对端内核缓冲区还没被应用 `read` 就崩溃，数据照样丢，必须靠应用层 ACK |
| 往已收到 `RST` 的连接 `write` 会怎样？ | 第一次写触发对端回 `RST` 时通常已返回成功；收到 `RST` 后的**第二次写**会触发内核发送 `SIGPIPE` 信号（默认终止进程），忽略信号则返回 `EPIPE` |
| 什么是路径 MTU 黑洞？ | 中间链路 MTU 小于两端协商值，包带 `DF` 不允许分片，而路由器的 ICMP 报错被防火墙拦截，表现为「小包全通、大包必挂」 |
| HTTP/2 解决了队头阻塞吗？ | 只解决了 HTTP/1.1 应用层的队头阻塞；所有 Stream 复用同一条 TCP 连接，一旦丢 1 个 TCP 包，内核按序交付会卡住所有 Stream。QUIC（HTTP/3）基于 UDP 按 Stream 独立排序才彻底解决 |
| 多路复用为什么必须配非阻塞 IO？ | `epoll` 返回可读只代表那一刻有事件；若 checksum 错误被丢弃或被别的线程读空，阻塞 `read` 会把整个事件循环线程挂死，见 [为什么 IO 多路复用必须搭配非阻塞 IO](/blog/2026/09/16/io-multiplexing-nonblocking/) |

## 总结

三次握手看双队列与 `tcp_abort_on_overflow`，四次挥手看谁先发 `FIN`（决定 `TIME_WAIT` 在哪端）以及应用调没调 `close`（决定有没有 `CLOSE_WAIT` 泄漏）。传输速率受 `min(cwnd, rwnd) / RTT` 约束：丢包靠 SACK 精准补洞、靠快恢复避免 `cwnd` 退回 1；而断电、拔网线和业务死锁，最终都得靠应用层自己的心跳与确认兜底。
