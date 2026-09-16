---
title: KCP 源码阅读：可靠 UDP 的 ARQ 与按序交付
date: 2026-09-15 20:40:00
categories: 网络
tags:
	- kcp
	- udp
	- arq
	- 源码阅读
---

丢一个包，KCP 的后续消息同样交付不了。这句话得放在最前面，因为「用 KCP 绕开 TCP 的队头阻塞」是关于它最流行的误读。`ikcp.c` 里把段从乱序缓存搬进可交付队列的条件只有一行：

```c
if (seg->sn == kcp->rcv_nxt && kcp->nrcv_que < kcp->rcv_wnd) { /* 搬 */ }
```

`sn` 对不上 `rcv_nxt` 就停在那里。一条 KCP 流内部是**严格按序交付**的，它只是把边界从字节换成了消息。要去掉队头阻塞得开多个 `conv`，或者在上面套一层多路复用（比如 smux）——那是另一层的事。

那 KCP 的 README 写的 "Average RTT reduce 30% - 40%" 是怎么来的？答案不浪漫：**它靠更激进的默认值和把拥塞控制的决定权交还给你**。RTO 下限从 Linux TCP 的 200ms 降到 30ms，退避从 ×2 改成 ×1.5，快重传门限从 3 个重复 ACK 降到 2 次跨越，甚至允许你一行调用把拥塞窗口整个关掉。丢包重传得更早、更频繁，延迟的长尾自然被削平；代价是同样的数据要多花带宽，而且在真的拥塞的链路上，你自己得替内核做本来由内核做的判断。

这笔交易在实时音视频、MOBA/FPS 的状态同步里划算：一帧操作 200ms 后到达就等于没到，宁可多发两份。在文件传输里就是纯亏。

写代码之前先把 KCP 的边界划清楚。本文对照 [skywind3000/kcp](https://github.com/skywind3000/kcp) 当前 master（`b1a7a21`，约 1400 行单文件 C）。

## 一、KCP 管什么，不管什么

| 能力 | 内核 TCP | KCP |
| --- | --- | --- |
| 连接建立 | SYN 三次握手 | **没有**。`conv` 由你带外协商或从首包学 |
| 收发包 | 内核 socket | 你的 `sendto`/`recvfrom`，通过 `kcp->output` 回调和 `ikcp_input` 喂进来 |
| 校验和 | 必需 | 没有。只能依赖外层 UDP 校验和（IPv4 下允许为 0） |
| 加密 | 无（靠 TLS） | 无 |
| 路径 MTU 发现 | PMTUD | 无。`ikcp_setmtu` 手填，默认 1400 |
| 定时器 | 内核时钟 | 你周期性调 `ikcp_update(kcp, now_ms)` |
| 保活 | `SO_KEEPALIVE` | 无。重传 20 次后只把 `kcp->state` 置 `(IUINT32)-1`，不通知任何人 |
| 拥塞控制 | CUBIC / BBR | 一个简化的 Reno，且可用 `nocwnd=1` 关掉 |
| 交付语义 | 字节流 | 默认保留消息边界，`stream=1` 才退化成流 |
| 流量控制单位 | 字节 | **包**（`wnd` 字段的单位是段数） |

所以 KCP 不是传输栈，是一个**纯内存的 ARQ 状态机**：你给它字节和时间，它告诉你该往网络上写什么、哪些字节可以交给应用。整个 `ikcpcb` 里没有一个 socket、没有一个线程、没有一次 `gettimeofday`。这个设计选择让它能塞进任何事件循环（也让后面那几个坑成为必然）。

## 二、一种段，四个字段管全部状态

KCP 只有一种包结构，数据和控制信令共用同一个 24 字节头（`IKCP_OVERHEAD = 24`）：

```text
0               4   5   6       8 (BYTE)
+---------------+---+---+-------+
|     conv      |cmd|frg|  wnd  |
+---------------+---+---+-------+   8
|      ts       |      sn       |
+---------------+---------------+  16
|      una      |      len      |
+---------------+---------------+  24
|        DATA (optional)        |
+-------------------------------+
```

| 字段 | 宽度 | 含义 |
| --- | --- | --- |
| `conv` | 4B | 会话号。**对不上整包丢弃**，`ikcp_input` 直接 `return -1` |
| `cmd` | 1B | `81` PUSH（数据）/ `82` ACK / `83` WASK（问窗口）/ `84` WINS（报窗口） |
| `frg` | 1B | 本段之后还剩几个分片，`0` 表示一条消息的最后一片 |
| `wnd` | 2B | 发送方此刻的剩余接收窗口 `rcv_wnd - nrcv_que`，单位是包 |
| `ts` | 4B | 发送时刻（ms）。ACK 原样抄回，用来量 RTT |
| `sn` | 4B | 段序号 |
| `una` | 4B | 发送方的 `rcv_nxt`，含义是「`una` 之前我全收齐了」 |
| `len` | 4B | `data` 长度 |

几个容易踩的推论：

- **`mss = mtu - 24 = 1376`**（默认）。`ikcp_send` 按 `mss` 切片，`frg` 从 `count-1` 倒着数到 `0`，这样接收端 `ikcp_peeksize` 看到队首段的 `frg` 就知道要凑几片。
- **单条消息上限 127 片**。`ikcp_send` 里 `if (count >= (int)IKCP_WND_RCV) return -2;`，`IKCP_WND_RCV` 是 128，所以默认最大约 174KB，且这个判断用的是常量而不是对端真实窗口。
- **`frg` 是 1 字节**，这也是分片数不能超过 255 的硬约束；128 的限制比它更紧，因为接收窗口至少要装得下一整条消息。
- **一个 UDP 包里可以有多个段**。`ikcp_flush` 往 `kcp->buffer` 里连续编码，攒够接近 `mtu` 才调一次 `output`，所以 `ikcp_input` 是个 `while (1)` 循环，逐段解码直到 `size` 耗尽。

真正维持可靠性的只有三个游标，值得单独记住：

- `snd_una`：最小的未确认序号。`snd_buf` 队首的 `sn` 就是它（`ikcp_shrink_buf`）。
- `snd_nxt`：下一个要分配的序号。`snd_nxt - snd_una` 就是在途包数。
- `rcv_nxt`：期待的下一个序号，也是发出去的所有段里填的 `una`。

## 三、四个队列和一次收包路径

`ikcpcb` 里有四条链表，发送和接收各两级。两级不是冗余，是把「应用的速度」和「网络的速度」解耦：

```text
                      发送侧                                       接收侧
  ikcp_send(buf, len)                                      ikcp_recv(buf, len)
        │ 按 mss 切片，frg = count-i-1                             ▲
        │                                                         │ 按 frg 把分片拼回一条消息
        ▼                                                         │
  ┌─────────────┐                                        ┌─────────────┐
  │  snd_queue  │ 无界！背压只能靠 ikcp_waitsnd          │  rcv_queue  │ 连续、可交付
  └─────────────┘                                        └─────────────┘
        │ ikcp_flush:                                              ▲
        │   while (snd_nxt < snd_una + cwnd)                       │ sn == rcv_nxt 才搬
        │   分配 sn，填 una / wnd / ts                             │ 搬一个 rcv_nxt++
        ▼                                                         │
  ┌─────────────┐                                        ┌─────────────┐
  │   snd_buf   │ 已发未确认，受窗口约束                 │   rcv_buf   │ 乱序缓存，按 sn 有序插入
  └─────────────┘                                        └─────────────┘
        │ kcp->output(buf, len, kcp, user)                         ▲
        ▼                                                         │ ikcp_input(data, size)
      你的 sendto  ─────────────  UDP  ─────────────  你的 recvfrom
```

`snd_queue` 到 `snd_buf` 的闸门是三个窗口取小：

```c
cwnd = _imin_(kcp->snd_wnd, kcp->rmt_wnd);
if (kcp->ccops != NULL || kcp->nocwnd == 0) cwnd = _imin_(kcp->cwnd, cwnd);
```

`snd_wnd` 是本端配置（默认 32），`rmt_wnd` 是对端最近一个包里报的 `wnd`，`cwnd` 是拥塞窗口。注意 `nocwnd=1` 只去掉第三项，前两项永远有效——**关掉拥塞控制不等于无限发**。

一次收包完整走下来是这样（`ikcp_input` → `ikcp_parse_data` → `ikcp_recv`）：

1. 校 `conv`，不匹配 `return -1`。
2. `kcp->rmt_wnd = wnd`：**每个段都更新**，包括 ACK 和乱序到达的旧包。
3. `ikcp_parse_una(una)`：把 `snd_buf` 里 `sn < una` 的段全部删掉；`ikcp_shrink_buf()` 重算 `snd_una`。这是累积确认，一个 `una` 能清掉一片。
4. 按 `cmd` 分支。PUSH 走第 5 步，ACK 走第 6 步。
5. `sn < rcv_nxt + rcv_wnd` 则 `ikcp_ack_push(sn, ts)` 把 ACK 记到待发列表——**即使 `sn < rcv_nxt`（重复包）也照样回 ACK**，否则对端会一直重传。落在窗口内且不是重复的才 `ikcp_parse_data` 插进 `rcv_buf`。
6. `ikcp_parse_ack(sn)` 删掉 `snd_buf` 里对应那一段；`ikcp_update_ack(rtt)` 更新 srtt/rttval/rto。
7. 循环结束后，用本次包里最大的那个 ACK 调一次 `ikcp_parse_fastack`，给所有 `sn` 更小且仍在 `snd_buf` 里的段 `fastack++`。
8. `snd_una` 前进了就调整 `cwnd`。
9. 应用调 `ikcp_recv`：从 `rcv_queue` 头部按 `frg` 拼出一条完整消息，然后**再跑一次 `rcv_buf → rcv_queue` 的搬运**（因为刚腾出了窗口空间），必要时置 `IKCP_ASK_TELL` 主动告知窗口已恢复。

第 7 步为什么要延后到循环外，是个值得说的细节。一个 UDP 包里可能捎回好几个 ACK，如果边解析边累加 `fastack`，同一个段会被同一个包重复计数好几次，快重传立刻被触发。把 `maxack` 攒到循环外统一处理，语义才是「收到了一个跨越 `sn` 的确认」。

## 四、`ikcp_flush`：三种重传触发条件

`ikcp_flush` 是整个协议的心脏，按顺序做五件事：回 ACK、探测窗口、开窗搬数据、发数据段、调窗口。第四步的循环是最该读的一段：

```c
/* ikcp_flush 片段，省略 ccops / pacing 分支，其余忠实于原文 */

// fastresend == 0 时，resent 是 0xffffffff，等于永不触发快重传
resent = (kcp->fastresend > 0)? (IUINT32)kcp->fastresend : 0xffffffff;
// 普通模式给首次重传额外加 1/8 RTO 的余量；nodelay 模式不加
rtomin = (kcp->nodelay == 0)? (kcp->rx_rto >> 3) : 0;

for (p = kcp->snd_buf.next; p != &kcp->snd_buf; p = p->next) {
    IKCPSEG *segment = iqueue_entry(p, IKCPSEG, node);
    int needsend = 0;

    if (segment->xmit == 0) {                 // 情况一：首发
        needsend = 1;
        segment->xmit++;
        segment->rto = kcp->rx_rto;           // 快照当时的 RTO，之后独立退避
        segment->resendts = current + segment->rto + rtomin;
    }
    else if (_itimediff(current, segment->resendts) >= 0) {   // 情况二：超时重传
        needsend = 1;
        segment->xmit++;
        kcp->xmit++;                          // 全局重传计数，唯一的健康度指标
        if (kcp->nodelay == 0) {
            // 普通模式：rto += max(rto, rx_rto)，约等于翻倍
            segment->rto += _imax_(segment->rto, (IUINT32)kcp->rx_rto);
        } else {
            // nodelay==1 → rto *= 1.5；nodelay>=2 → 只加 rx_rto/2，退避更保守
            IINT32 step = (kcp->nodelay < 2)? ((IINT32)(segment->rto)) : kcp->rx_rto;
            segment->rto += step / 2;
        }
        segment->resendts = current + segment->rto;
        lost = 1;                             // 本轮发生超时 → 稍后重置 cwnd = 1
    }
    else if (segment->fastack >= resent) {    // 情况三：快重传
        // fastlimit 限制一个段最多被快重传几次（默认 5），避免单段反复抢带宽
        if ((int)segment->xmit <= kcp->fastlimit || kcp->fastlimit <= 0) {
            needsend = 1;
            segment->xmit++;
            segment->fastack = 0;
            segment->resendts = current + segment->rto;   // 注意：不退避 rto
            change++;                         // 本轮发生快重传 → 稍后降 ssthresh
        }
    }

    if (needsend) {
        segment->ts = current;                // 每次发都刷 ts，RTT 量的是最后一次
        segment->wnd = seg.wnd;               // 搭车带上本端剩余窗口
        segment->una = kcp->rcv_nxt;          // 搭车带上累积确认

        size = (int)(ptr - buffer);
        if (size + IKCP_OVERHEAD + segment->len > (int)kcp->mtu) {
            ikcp_output(kcp, buffer, size);   // 攒满一个 MTU 才真正下发
            ptr = buffer;
        }
        ptr = ikcp_encode_seg(ptr, segment);
        if (segment->len > 0) {
            memcpy(ptr, segment->data, segment->len);
            ptr += segment->len;
        }

        if (segment->xmit >= kcp->dead_link) {
            kcp->state = (IUINT32)-1;         // 只置位，不回调、不断开
        }
    }
}
```

读这段要抓住的点：

- **每个段自带 `rto`，不是全局的。** `segment->rto` 在首发时从 `kcp->rx_rto` 快照一份，之后独立退避。所以同一时刻 `snd_buf` 里不同段的超时阈值可以差很多倍，一条链路上「老段退避到 2s、新段还是 100ms」是正常状态。
- **快重传不退避。** 情况三只把 `resendts` 推后一个 `rto`，`rto` 本身不变。这是「我确信它丢了，不是网络慢」的判断，符合直觉，但也意味着判断错了的话会持续放大流量——所以有 `fastlimit`。
- **`ts` 每次重发都被刷新。** 配合 `ikcp_update_ack` 只在 `current >= ts` 时才更新 RTT，重传段回来的 ACK 量到的是最后一次发送到 ACK 的时间，不会把重传延迟算进 srtt。
- **`rtomin` 是普通模式和极速模式的第一道分水岭。** 普通模式首次超时要多等 `rx_rto/8`，`rx_rto` 初值 200ms 就是 25ms 的额外延迟；nodelay 模式直接砍掉。
- **`change` 和 `lost` 只是标记**，真正调窗口在函数末尾：

```c
/* 原文用 if 判断下限，这里用 _imax_ 改写，语义一致 */
if (change) {   // 快重传：ssthresh 减半，cwnd 降到 ssthresh + resent
    IUINT32 inflight = kcp->snd_nxt - kcp->snd_una;
    kcp->ssthresh = _imax_(inflight / 2, IKCP_THRESH_MIN);
    kcp->cwnd = kcp->ssthresh + resent;
    kcp->incr = kcp->cwnd * kcp->mss;
}
if (lost) {     // 超时：直接砍到 1，等价 TCP 的 RTO timeout
    kcp->ssthresh = _imax_(prior_cwnd / 2, IKCP_THRESH_MIN);
    kcp->cwnd = 1;
    kcp->incr = kcp->mss;
}
if (kcp->cwnd < 1) { kcp->cwnd = 1; kcp->incr = kcp->mss; }
```

最后那个兜底有个副作用值得知道：`ikcp_create` 里 `cwnd = 0`，而 `cwnd` 是在函数**开头**参与开窗计算的。所以默认模式下第一次 `flush` 搬不出任何数据，`cwnd` 在函数末尾才被抬到 1，真正开始发要等到下一次 `flush`——默认 `interval=100` 就是握手后一次性多出来的 100ms。极速模式（`nocwnd=1`）绕过 `cwnd` 判断，没有这个问题。

`ssthresh` 初值 `IKCP_THRESH_INIT = 2` 也说明 KCP 基本不做慢启动：`cwnd` 从 0 涨到 2 就进入拥塞避免阶段了。

## 五、ACK 的接收侧：`ikcp_input` 的 ACK 分支

```c
if (cmd == IKCP_CMD_ACK) {
    // ts 是我们发出去时填的，原样回来了，差值就是 RTT
    if (_itimediff(kcp->current, ts) >= 0) {
        ikcp_update_ack(kcp, _itimediff(kcp->current, ts));
    }
    ikcp_parse_ack(kcp, sn);      // 精确删掉 snd_buf 里 sn 这一段
    ikcp_shrink_buf(kcp);         // 队首变了，重算 snd_una

    if (flag == 0) {              // 本次 ikcp_input 里第一个 ACK
        flag = 1;
        maxack = sn;
        latest_ts = ts;
    } else if (_itimediff(sn, maxack) > 0) {
        #ifndef IKCP_FASTACK_CONSERVE
            maxack = sn; latest_ts = ts;
        #else
            // 保守模式：还要求这个 ACK 对应的发送时刻更晚
            if (_itimediff(ts, latest_ts) > 0) { maxack = sn; latest_ts = ts; }
        #endif
    }
}
/* ... 循环结束后 ... */
if (flag != 0) {
    ikcp_parse_fastack(kcp, maxack, latest_ts);
}
```

`ikcp_parse_fastack` 的循环体是快重传的全部依据：

```c
for (p = kcp->snd_buf.next; p != &kcp->snd_buf; p = next) {
    IKCPSEG *seg = iqueue_entry(p, IKCPSEG, node);
    next = p->next;
    if (_itimediff(sn, seg->sn) < 0) break;    // snd_buf 按 sn 有序，可以早退
    else if (sn != seg->sn) {
        #ifndef IKCP_FASTACK_CONSERVE
            seg->fastack++;
        #else
            if (_itimediff(ts, seg->ts) >= 0) seg->fastack++;
        #endif
    }
}
```

`IKCP_FASTACK_CONSERVE` 在当前 master 里是**默认打开的**（`ikcp.c` 第 20 行）。它多要求一个条件：只有当这个 ACK 确认的段「发得比你晚」时，才算你被跨越了一次。

为什么需要这个条件：`seg->ts` 每次重发都会刷新。假设段 5 重传过、`ts` 已经很新，这时一个确认段 8 的 ACK 回来了，而段 8 是更早发出去的。不加保守判断，段 5 的 `fastack` 会被 +1，而这个跨越其实只说明「8 比 5 的旧版本先到」，跟 5 的重传版本是否丢失无关。关掉 `CONSERVE` 在重传频繁的链路上会明显放大重传量。

对比一下 TCP 的 `fastresend` 门限：TCP 要 3 个重复 ACK（现代内核更多用 RACK 走时间维度），KCP 官方推荐 `resend=2`。少一次跨越就重传，早 1 个 RTT 修复丢包，也更容易误判。

## 六、时钟：`ikcp_update` / `ikcp_flush` / `ikcp_check`

KCP 没有自己的时间源，`kcp->current` 完全由 `ikcp_update` 的参数喂进来。三个函数的分工：

```c
void ikcp_update(ikcpcb *kcp, IUINT32 current)
{
    kcp->current = current;
    if (kcp->updated == 0) {                  // 第一次调用才对齐 ts_flush
        kcp->updated = 1;
        kcp->ts_flush = kcp->current;
    }
    slap = _itimediff(kcp->current, kcp->ts_flush);
    if (slap >= 10000 || slap < -10000) {     // 时钟跳变超过 10s：重新对齐
        kcp->ts_flush = kcp->current;
        slap = 0;
    }
    if (slap >= 0) {                          // 到点了
        kcp->ts_flush += kcp->interval;
        if (_itimediff(kcp->current, kcp->ts_flush) >= 0)
            kcp->ts_flush = kcp->current + kcp->interval;
        ikcp_flush(kcp);
    }
}
```

关键的一行在 `ikcp_flush` 开头：

```c
if (kcp->updated == 0) return;
```

**`ikcp_update` 一次都没调过，`ikcp_flush` 直接返回，什么都不做。** 于是 `ikcp_send` 的数据永远停在 `snd_queue`，`ikcp_ack_push` 攒下的 ACK 永远不发出去，对端一路重传到 `dead_link`。这是接手 KCP 的人第一个会踩的坑，后面单独说排查方式。

`ikcp_check` 是给「不想每 10ms 空转一次」的人准备的：

```c
IUINT32 ikcp_check(const ikcpcb *kcp, IUINT32 current)
{
    if (kcp->updated == 0) return current;                  // 还没启动，立刻调
    if (_itimediff(current, ts_flush) >= 0) return current; // flush 已到点
    tm_flush = _itimediff(ts_flush, current);
    for (遍历 snd_buf) {
        diff = _itimediff(seg->resendts, current);
        if (diff <= 0) return current;                      // 有段该重传了
        if (diff < tm_packet) tm_packet = diff;
    }
    minimal = _imin_(tm_packet, tm_flush);
    if (minimal >= kcp->interval) minimal = kcp->interval;  // 上限是 interval
    return current + minimal;
}
```

它返回的是「下次最晚该调 `ikcp_update` 的时刻」。要点有两个：

- 它是 **O(snd_buf)** 的线性扫描。管理上万条连接时，`ikcp_check` 本身可能比空转 `ikcp_update` 更贵。真要做大规模调度，把每个连接的下次到期时间挂到一个层次化时间轮上，参考 [时间轮算法：原理、层次化实现和一组 Go Timer 接口](/blog/2026/08/28/timing-wheel/)。
- 它的语义是「在此之前没有 `ikcp_send`/`ikcp_input`」。一旦你调了这两个函数，之前算出的时刻就作废了，必须重新 `ikcp_check`。

顺带一个源码里的裂缝：`ikcp_interval()` 在 `ikcp.c` 里有定义，但 `ikcp.h` 里**没有声明**。想单独改 `interval` 只能走 `ikcp_nodelay(kcp, -1, interval, -1, -1)`（负数表示不改该项），或者自己补声明。

## 七、`nodelay` 到底改了什么

`ikcp_nodelay(kcp, nodelay, interval, resend, nc)` 只是四个赋值，但每个都对应上面某段代码的一条分支：

| 参数 | 普通模式 `(0, 40, 0, 0)` | 极速模式 `(1, 10, 2, 1)` | 源码里的影响 |
| --- | --- | --- | --- |
| `nodelay` | 0 | 1 | `rx_minrto`：100ms → 30ms；`rtomin`：`rx_rto/8` → 0；RTO 退避：×2 → ×1.5 |
| `interval` | 40ms | 10ms | `flush` 周期，即 ACK 延迟上限和重传检查粒度，被夹在 `[10, 5000]` |
| `resend` | 0（关闭） | 2 | `resent`：`0xffffffff` → 2，快重传门限 |
| `nc` | 0 | 1 | `nocwnd`：开窗时不再 `min(cwnd, ...)`，丢包不降速 |

`interval` 的影响常被低估。KCP 的 ACK 不是收到就发，是攒在 `acklist` 里等下一次 `flush`——`interval=100` 意味着 ACK 最多晚 100ms 出门，直接算进对端测到的 RTT，进而抬高它的 RTO。把 `interval` 从 100 降到 10，光这一项就能砍掉几十毫秒的虚假 RTT。

另外几个常量顺手记住（都在 `ikcp.c` 顶部）：

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `IKCP_MTU_DEF` | 1400 | UDP 载荷上限，`mss = 1376` |
| `IKCP_WND_SND` | 32 | `snd_wnd` 默认值 |
| `IKCP_WND_RCV` | 128 | `rcv_wnd` 默认值，且是它的**下限** |
| `IKCP_RTO_DEF` / `MIN` / `NDL` / `MAX` | 200 / 100 / 30 / 60000 | ms |
| `IKCP_THRESH_INIT` | 2 | `ssthresh` 初值，几乎等于没有慢启动 |
| `IKCP_FASTACK_LIMIT` | 5 | `fastlimit`，单段快重传次数上限 |
| `IKCP_DEADLINK` | 20 | 单段重传 20 次后置 `state = -1` |
| `IKCP_PROBE_INIT` / `LIMIT` | 5000 / 120000 | 零窗口探测的起步和上限间隔（ms） |
| `IKCP_ACK_FAST` | 3 | 历史遗留，当前代码里**没有任何地方引用** |

`README.md` 里写「最大发送窗口和最大接收窗口……默认为 32」，和代码已经不一致了：`ikcp_wndsize` 对接收窗口做的是 `kcp->rcv_wnd = _imax_(rcvwnd, IKCP_WND_RCV)`，你传 32 也会被抬到 128。头文件里 `// set maximum window size: sndwnd=32, rcvwnd=32 by default` 同样是过时注释。读 KCP 的文档不如读它的 `const`。

## 八、Go 侧长什么样

`ikcp.c` 是协议本体，工程里几乎没人直接裸用它——UDP 收发、连接管理、加密、FEC 都得自己补。Go 生态里 [xtaci/kcp-go](https://github.com/xtaci/kcp-go) 把这层包掉了，**它提供的是 `net.Conn`，不是协议本体**：

```go
// 客户端：block=nil 不加密，dataShards=10 parityShards=3 是 Reed-Solomon FEC
// 这两项都是 kcp-go 加在 KCP 之外的，upstream C 实现里没有
sess, err := kcp.DialWithOptions("game.example.com:29900", nil, 10, 3)
if err != nil {
	return err
}
defer sess.Close()

sess.SetStreamMode(true)          // 对应 kcp->stream = 1，丢掉消息边界换取填满 mss
sess.SetNoDelay(1, 10, 2, 1)      // 就是 ikcp_nodelay，参数一一对应
sess.SetWindowSize(128, 512)      // 对应 ikcp_wndsize，单位是包不是字节
sess.SetMtu(1200)                 // 留余量给 FEC 头 + 加密头 + 外层 IP/UDP
sess.SetACKNoDelay(true)          // kcp-go 的扩展：收到包立刻 flush 一次 ACK

// 服务端
lis, err := kcp.ListenWithOptions(":29900", nil, 10, 3)
for {
	conn, err := lis.AcceptKCP()  // 返回 *UDPSession，可以继续调上面那组 Set
	if err != nil {
		return err
	}
	go handle(conn)
}
```

两个包装层面的事实，读源码才会知道：

- **`conv` 是客户端随机生成、服务端从首包学的。** `DialWithOptions` 里 `binary.Read(rand.Reader, binary.LittleEndian, &convid)`；服务端 `sess.go` 里按 `addr.String()` 在 map 里找会话，找不到就用包头前 4 字节的 `conv` 新建一个。**解复用的键是 remote addr，不是 `conv`**，`conv` 只用来检测「同一个地址换了会话」。这也解释了为什么 KCP 协议本身不需要握手。
- **`UDPSession` 自带一把 `sync.Mutex`。** 所有进 `ikcpcb` 的调用都在锁内，因为 `ikcpcb` 本身完全没有同步。

如果你在设计自己的封装层，会发现要补的东西和写一个 RPC 框架时要补的那堆基础设施高度重合——分帧、会话、超时语义，见 [RPC 原理、Go net/rpc 与一份极简实现](/blog/2026/09/04/rpc-from-scratch/)。取消和超时的传播则最好从一开始就交给 `context`，见 [Go context：设计、源码与代价](/blog/2026/09/02/golang-context/)。

## 九、避坑

### 1. `conv` 不一致：静默丢包，零日志

`ikcp_input` 的第一件事：

```c
data = ikcp_decode32u(data, &conv);
if (conv != kcp->conv) return -1;
```

没有日志、没有回执、没有计数器。表现是一端 `ikcp_send` 返回正数、`output` 回调也在正常触发、`tcpdump` 能抓到包，但另一端 `ikcp_recv` 永远返回 `-1`。

这个坑的高发场景是：服务端按 remote addr 建会话，但 `conv` 用了自己生成的随机数，而不是从客户端首包里读出来的那个。或者两端对 `conv` 的字节序理解不一致（KCP 用小端）。

排查三步：

1. **先看 `ikcp_input` 的返回值。** 绝大多数集成代码写成 `ikcp_input(kcp, buf, n);`，返回值直接丢掉。加一行日志区分 `-1`（conv 不匹配）、`-2`（`len` 字段和实际长度不符，通常是外层封装少截了字节）、`-3`（`cmd` 非法，通常是把非 KCP 包喂进来了，比如 STUN 响应或 FEC 包没剥头）。
2. **用 `ikcp_getconv(ptr)` 解出收到包的 `conv`**，和本端 `kcp->conv` 打在一起比。这个函数就是为此存在的。
3. **确认 `conv` 的协商路径**。KCP 不管握手，所以这个值一定来自你的某个设计决定：登录响应下发、从首包学、或者约定为「高 16 位客户端索引 + 低 16 位服务端索引」（`protocol.txt` 推荐的做法）。把这条链路画出来，错在哪一目了然。

### 2. 忘了周期性 `ikcp_update`：连 ACK 都不发

前面提过 `ikcp_flush` 开头的 `if (kcp->updated == 0) return;`。更隐蔽的变体是**调了但调得不够勤**：只在 `ikcp_send` 之后调一次、或者挂在一个被业务逻辑阻塞的循环里。

这个故障的诊断特征很清楚：

- **单向失效**：发得出去（`ikcp_send` 把数据放进队列，某次 `flush` 发了首发包），但收不到 ACK，因为 ACK 要靠对端的 `flush` 发、重传要靠本端的 `flush` 做。
- **`kcp->xmit` 疯涨**：这是唯一的全局重传计数器，只在超时重传路径 `kcp->xmit++`。把它和「累计发送段数」的比值打成监控指标，健康链路应该在百分之几，忘记 `update` 的一端会看到对面的重传比接近 100%。
- **20 次之后彻底静默**：`segment->xmit >= kcp->dead_link` 把 `state` 置成 `(IUINT32)-1`，然后……什么也不会发生。KCP 不会回调、不会让 `ikcp_send` 开始报错。**你必须自己轮询 `kcp->state`**，否则这条连接会以「一切正常但没有数据」的形态挂着。

正确姿势是一个独立的定时器驱动所有 KCP 实例，`interval` 多少就至少调多勤，或者用 `ikcp_check` 精确调度。别把 `ikcp_update` 和业务处理放在同一个可能阻塞的调用栈上。

### 3. `rmt_wnd == 0`：探测起步 5 秒，指数退避到 2 分钟

对端的接收队列满了（应用不读、或者读得比收得慢），它报回来的 `wnd` 就是 0。本端 `cwnd = _imin_(snd_wnd, rmt_wnd)` 变成 0，一个字节都发不出去。此时只能靠 WASK 探测：

```c
if (kcp->rmt_wnd == 0) {
    if (kcp->probe_wait == 0) {
        kcp->probe_wait = IKCP_PROBE_INIT;            // 5000ms
        kcp->ts_probe = kcp->current + kcp->probe_wait;
    } else if (_itimediff(kcp->current, kcp->ts_probe) >= 0) {
        if (kcp->probe_wait < IKCP_PROBE_INIT)
            kcp->probe_wait = IKCP_PROBE_INIT;
        kcp->probe_wait += kcp->probe_wait / 2;       // ×1.5 退避
        if (kcp->probe_wait > IKCP_PROBE_LIMIT)       // 上限 120000ms
            kcp->probe_wait = IKCP_PROBE_LIMIT;
        kcp->ts_probe = kcp->current + kcp->probe_wait;
        kcp->probe |= IKCP_ASK_SEND;                  // 下次 flush 发 WASK
    }
} else {
    kcp->ts_probe = 0; kcp->probe_wait = 0;
}
```

注意第一次探测要等整整 5 秒（源码注释写的 "7 secs to probe window size" 和值对不上，是残留注释），之后 7.5s、11.25s……最坏 2 分钟一次。

反向恢复路径靠对端主动：`ikcp_recv` 里如果发现「进来时队列是满的、现在腾出空间了」，会置 `probe |= IKCP_ASK_TELL`，下次 `flush` 主动发一个 WINS 报窗口。所以正常情况下恢复是即时的，5 秒探测只是兜底。

真正会咬人的是**兜底路径被触发**：那个 WINS 包丢了，于是两端一个在等窗口、一个以为已经通知过，卡到下一次 WASK。表现是业务侧看到规律性的 5s / 7.5s 级别卡顿。

防御手段只有一个方向：**别让接收队列满**。

- 应用侧保证 `ikcp_recv` 被及时调用，不要在收包回调里做重活。
- 发送侧用 `ikcp_waitsnd()`（返回 `nsnd_buf + nsnd_que`）做背压。`snd_queue` 是无界链表，`ikcp_send` 除了分片数超限几乎不会失败，所以**背压完全是调用方的责任**。一个常见阈值是 `2 * snd_wnd`，超了就丢低优先级数据或者断开。
- `rcv_wnd` 设大一点。它的代价只是内存（`rcv_wnd * mss` 字节的乱序缓存），128 个包在现在的机器上不值得省。

### 4. `nodelay=1` + `nc=1` 在差网络上会把链路打满

极速模式 `ikcp_nodelay(kcp, 1, 10, 2, 1)` 的四项叠在一起是这样的效果：

- `rx_minrto = 30`，RTO 下限 30ms。真实 RTT 60ms 的链路上，一次乱序就足以触发超时重传。
- `rtomin = 0`，首次超时判定不留余量。
- RTO 退避只有 ×1.5，而 `lost` 分支同时把 `cwnd` 砍到 1——但 `nocwnd=1` 让开窗计算根本不看 `cwnd`，所以这个「砍到 1」是**空操作**。
- `resend=2`，两次跨越就重传。

结果：链路真的开始丢包时，KCP 唯一还在生效的限速是 `min(snd_wnd, rmt_wnd)`，而它不随丢包收缩。你会看到一条 20% 丢包的链路上，实际发送字节是有效数据的 3~5 倍——重传引起更多拥塞，拥塞引起更多重传。这不是 KCP 的 bug，是你把拥塞控制关掉之后本该自己接手的责任。

判断是否踩中，看两个数：

1. `kcp->xmit`（超时重传次数）除以累计发出的段数。稳定超过 20% 就说明重传在放大，不是偶发丢包。
2. `output` 回调里累加的字节数除以 `ikcp_send` 传入的字节数。这个比值是真实的带宽放大倍数，健康值在 1.1 上下。

改法按激进程度递减：

- **`nc=0`，保留拥塞控制，只留 `nodelay=1` 和小 `interval`。** 大多数场景这就够了，延迟收益主要来自 `rx_minrto` 和 `interval`，不是来自关拥塞。
- **`snd_wnd` 调小。** 关掉 `cwnd` 后它是唯一的闸门，把它当成手工设定的 BDP：`snd_wnd ≈ 带宽(B/s) × RTT(s) / mss`。默认 32 在 1376 字节 mss 下约等于 44KB 在途，对应 50ms RTT 是 7Mbps。别不算就往上调到 1024。
- **`nodelay=2`。** 这个取值在文档里几乎没提，但源码里它让 RTO 退避变成 `rto += rx_rto / 2`，比 `nodelay=1` 的 `rto *= 1.5` 更克制，是「要低延迟但不想完全不退避」的中间档。
- **上 `ikcp_setcc` 挂自己的拥塞控制。** master 新增的 `IKCPOPS` 把 `on_ack` / `on_timeout` / `on_fast_retransmit` / `pacing_rate` 都开放出来了。但注意一个语义变化：装了 `ccops` 之后，`nocwnd` 会被忽略（`if (kcp->ccops != NULL || kcp->nocwnd == 0)`），`cwnd` 始终参与开窗。这套接口还很新，上生产前自己压测。

### 5. 两个短的：MTU 和并发

**`mtu` 是 UDP 载荷上限，不含外层。** `ikcp_setmtu(kcp, 1400)` 之后，`flush` 攒到接近 1400 字节才下发，实际以太网帧是 1400 + 8(UDP) + 20(IP) = 1428。在标准 1500 MTU 的路径上没问题，但 PPPoE（1492）、各类 VPN/隧道（1400 或更低）会触发 IP 分片。UDP 的 IP 分片比 TCP 危险得多：任一分片丢失整个数据报报废，而 KCP 看到的是「一整个包丢了」，会触发重传，于是重传的包又被分片。叠上 kcp-go 的 FEC 头和加密头就更紧。**跨公网建议 `mtu` 设到 1200 或更低**，牺牲一点头部开销换掉分片。另外 `wnd` 字段只有 16 位，`rcv_wnd` 设到 65536 以上会在编码时截断，这是个静默错误。

**`ikcpcb` 是纯单线程状态机。** 里面没有一把锁、一个原子变量。`ikcp_send` 和 `ikcp_input` 并发调用会同时改 `snd_buf` / `rcv_buf` 的链表指针和 `nsnd_buf` 计数，链表断裂之后的表现是 `iqueue` 遍历走进野指针——崩溃点在 `ikcp_flush` 里，和真正的错误现场（另一个线程的 `ikcp_send`）毫无关系，非常难查。要么把一个连接的所有 KCP 调用（含 `ikcp_update`）收敛到同一个线程/事件循环，要么像 kcp-go 那样在外面加一把覆盖全部入口的互斥锁。**不要试图只锁一部分入口**。

## 收束

KCP 是一个 1400 行的 ARQ 状态机：四条链表、三个游标、一个 24 字节头，靠 `ikcp_update` 周期性调用推进。它的「快」不来自更聪明的算法，而来自更小的 RTO 下限、更低的快重传门限、更短的 flush 间隔，以及允许你把拥塞控制整个关掉——前三项在大多数场景里安全且值得，第四项是把内核的责任转移到了你的肩上。

真正需要防的不是协议本身，是它刻意留白的地方：`conv` 要你协商、时钟要你驱动、背压要你用 `ikcp_waitsnd` 做、`state` 要你轮询、并发要你在外面锁。这些留白让它能跑在任何事件循环里，也让集成 KCP 出的故障基本都是集成的问题，而不是 ARQ 的问题。

选型上记住那条最初的判断：它保留消息边界，但一条流内部依然严格按序交付。如果你真正要解决的是队头阻塞，需要的是多路复用或多流，不是把 TCP 换成 KCP。
