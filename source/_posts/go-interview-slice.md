---
title: Go 面试：slice 的头部、扩容与共享底层数组
date: 2026-10-27 21:10:00
categories: Golang
tags:
	- golang
	- 面试
	- slice
---

面试里把 slice 叫成动态数组，下一问就是扩容倍数。要答的不是倍数，是三件事：slice 只是三个字的头，`append` 可能换掉底层数组，两个头指向同一块数组时，写其中一个会改到另一个。

这是 Go 面试系列的第一篇。同系列还有 [Go 面试：map 的 Swiss Table、扩容与并发](/blog/2026/10/27/go-interview-map/)、[Go 面试：channel 的阻塞、关闭与 select](/blog/2026/10/27/go-interview-channel/)。扩容数字以 Go 1.27 的 `runtime.nextslicecap` 为准。

## 头和数组是两样东西

slice 的值是三个字段：指向数组的指针、`len`、`cap`。赋值、传参复制的是这个头，不是数组。

```text
s 的头                         底层数组
+--------+-----+-----+        +---+---+---+---+---+---+
| array  | len | cap | -----> | 1 | 2 | 3 | 4 | _ | _ |
+--------+-----+-----+        +---+---+---+---+---+---+
         len=4  cap=6

a := s[1:3]
+--------+-----+-----+
| array+1|  2  |  5  | -----> 从 2 开始，cap 还剩 5
+--------+-----+-----+
```

`a` 和 `s` 的头各有一份，数组只有一块。`len` 是能读的元素个数，`cap` 是从 `array` 这个起点往后、到这块分配末尾还能放多少。切的起点越靠后，`cap` 越小，因为容量从新起点算，不从原数组开头算。

`nil` slice 的指针是 nil，`len` 和 `cap` 都是 0。`append` 可以作用在 nil 上。`[]int{}` 也是 `len` 0，但它不是 nil。`encoding/json` 把前者编成 `null`，后者编成 `[]`。

## 扩容：先算个数，再对齐到 size class

`append` 之后的 `len` 超过 `cap` 才分配新数组，把旧的 `[0, oldLen)` 拷过去。Go 1.27 的个数由 `nextslicecap` 决定：

```go
func nextslicecap(newLen, oldCap int) int {
    newcap := oldCap
    doublecap := newcap + newcap
    if newLen > doublecap {
        return newLen // 一次加很多，直接给到够用
    }
    const threshold = 256
    if oldCap < threshold {
        return doublecap // 小切片翻倍
    }
    for {
        // 从 2 倍平滑过渡到大约 1.25 倍
        newcap += (newcap + 3*threshold) >> 2
        if uint(newcap) >= uint(newLen) {
            break
        }
    }
    if newcap <= 0 { // 溢出
        return newLen
    }
    return newcap
}
```

`oldCap < 256` 才是「翻倍」。到了 256，增量是 `(newcap + 768) >> 2`，大约 1.25 倍再多一点，不是突然改成 1.25。一次 `append` 很多元素、超过两倍时，不翻倍，直接要 `newLen`。

这还不是你打印出来的 `cap`。`growslice` 把元素个数乘上元素大小，再向上对齐到 malloc 的 size class，多出来的位置折回元素个数。所以公式给出 832 个 `int`，堆上可能看到 848。

下面是 Go 1.27、`windows/amd64`、slice 逃逸到堆上时，`[]int` 每次变大的 `cap`：

| len 刚越过旧 cap | 观察到的 cap | 公式直接给出 |
| --- | --- | --- |
| 1, 2, 3, 5, 9, 17, 33, 65, 129 | 1, 2, 4, 8, 16, 32, 64, 128, 256 | 同左，翻倍 |
| 257 | 512 | 512 |
| 513 | 848 | 832，对齐后变成 848 |
| 849 | 1280 | 1252，对齐后变成 1280 |
| 1281 | 1792 | 1792 |
| 1793 | 2560 | 2432，对齐后变成 2560 |

元素是 `byte` 时更明显：第一个元素的公式结果是 1，对齐之后 `cap` 直接是 8。背「1, 2, 4, 8」只能对上某些元素宽度、而且切片必须在堆上。不逃逸的小切片在 Go 1.27 可能走 `growsliceBuf`，按栈缓冲区的 size class 抬容量，和这张表不是同一列数。

## 共享数组时 append 会改到别人

`s[:2]` 的 `cap` 仍是 `s` 的 `cap`。`append` 发现还放得下，就写进同一块数组，不分配。

```go
package main

import "fmt"

func main() {
    s := []int{1, 2, 3, 4}

    a := s[:2]
    a = append(a, 9)
    fmt.Println(s, a) // [1 2 9 4] [1 2 9]

    b := s[:2:2] // 第三个下标把 cap 压到 2
    b = append(b, 7)
    fmt.Println(s[2], b) // 9 [1 2 7]，b 已经换了数组
}
```

三下标形式 `s[i:j:k]` 的 `cap` 是 `k-i`。把 `k` 设成 `j`，这段切片就不能再往后写进原数组。函数如果只想借用一段数据、又可能 `append`，入口先做这一步，或者 `append([]T(nil), in...)` 拷一份。

`append` 的结果必须接住。容量不够时新头在返回值里，丢掉它，调用方还拿着旧头。容量够时更糟：函数里的 `append` 已经改了调用方的数组，调用方的 `len` 却没变，多出来的元素用下标看不见，之后自己的 `append` 会把它盖掉。

## 避坑

- **大切小用，整块还活着。** `buf := readAll(); line := buf[:n]` 之后只要 `line` 还被引用，`buf` 那几百 KB 就回收不了。要留下的是拷贝：`append([]byte(nil), buf[:n]...)`。
- **`append` 在循环里传给别人。** `go func() { sink = append(s, x) }()` 这种写法，循环变量和共享数组叠在一起，goroutine 看到的既可能是旧头，也可能和下一轮写同一块数组。把当次的头当参数传进去。
- **用 `cap` 当已有数据。** `s[:cap(s)]` 会把没写过的尾部算进 `len`。尾部在含指针的元素上是零值，在不含指针、刚扩完的那一段也是清过的，但它不是你的元素。长度只信 `len`。

## 面试重点

| 追问 | 答到这里可以停 |
| --- | --- |
| slice 是什么 | 三个字的头：指针、`len`、`cap`。传参复制头 |
| `len` 和 `cap` | `len` 是可读个数。`cap` 从当前指针算到分配末尾，不是从原数组开头算 |
| 什么时候扩容 | `append` 后的长度超过 `cap`。小于 256 翻倍，之后按 `(cap + 3×256) >> 2` 往上加，再对齐 size class |
| 两个 slice 互相影响 | 头可以不同，数组可以是同一块。`s[:i]` 仍可能写进 `s[i:]` |
| 怎么切断共享 | `s[i:j:j]`，或拷贝一份新数组 |
| `nil` 和空切片 | 都能 `append`。`nil` 的指针是 nil，JSON 分别是 `null` 和 `[]` |
| 扩容之后旧切片 | 还指着旧数组。没接到返回值的那一方看不到新元素 |

背倍数之前先说清：倍数作用在元素个数上，打印出来的 `cap` 还要过一遍 size class，而且切片在不在堆上，走的不是同一个函数。

## 总结

slice 的成本不在「变长」，在头和数组分离。`append` 在容量内是一次写入，容量外是一次分配加拷贝。面试里把共享数组、三下标切法和「返回值必须接住」说清楚，比背到第几档是 1.25 倍更有用。
