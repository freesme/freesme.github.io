---
layout: post
title: go httpfast
date: 2024-11-22 18:15 +0800
categories: [TIL]
---
### 来自官方的简介

https://github.com/valyala/fasthttp

简而言之，fasthttp 服务器比 net/http 快高达 10 倍。
**fasthttp 可能不适合你！**`fasthttp` 是为一些高性能边缘情况而设计的。除非您的服务器/客户端需要**每秒处理数千个中小型请求**，并且需要一致的低毫秒响应时间，否则 fasthttp 可能不适合您。**在大多数情况下，`net/http`它要好得多，**因为它(`net/http`)更易于使用并且可以处理更多情况。在大多数情况下，您甚至不会注意到性能差异。

### 多核系统的优化技巧

- 使用[“可重用端口”](https://pkg.go.dev/github.com/valyala/fasthttp/reuseport)监听器。通过允许多个进程或线程监听相同的端口，提高服务器对并发连接的处理能力，充分利用多核 CPU 的性能。
- 每个 CPU 核心运行一个单独的服务器实例，设置 GOMAXPROCS=1（避免多个核心之间的锁竞争，提高单个实例的处理效率）。
- 使用[taskset](http://linux.die.net/man/1/taskset)将每个服务器实例固定到单独的 CPU 核心上，确保每个实例在独立的核心上运行，减少上下文切换的开销，提高缓存命中率。
- 确保多队列网卡的中断在 CPU 核心之间均匀分布。有关详细信息，请参阅[这篇文章](https://blog.cloudflare.com/how-to-achieve-low-latency/)。
- 使用最新版本的 Go，因为每个版本都包含性能改进。

### fasthttp最佳实践

- 不要分配对象和`[]byte`缓冲区 —— 尽可能重用它们。Fasthttp API 设计鼓励这样做。
- [sync.Pool](https://pkg.go.dev/sync#Pool)是你最好的伙伴。



- [在生产环境中分析你的程序](http://go.dev/blog/pprof)
  `go tool pprof --alloc_objects your-program mem.pprof`
  通常比`go tool pprof your-program cpu.pprof`提供更好的优化机会洞察。
- 为热门路径编写[测试和基准测试](https://pkg.go.dev/testing)。
- 避免在`[]byte`和`string`之间进行转换，因为这可能导致内存分配和复制。Fasthttp API 为`[]byte`和`string`都提供了函数 —— 使用这些函数而不是在`[]byte`和`string`之间手动转换。有一些例外情况 —— 查看[这个维基页面](https://github.com/golang/go/wiki/CompilerOptimizations#string-and-byte)以获取更多详细信息。
- 定期在[race detector](https://go.dev/doc/articles/race_detector.html)下验证你的测试和生产代码。
- 在Web服务中优先使用[quicktemplate](https://github.com/valyala/quicktemplate)而不是[html/template](https://pkg.go.dev/html/template)。

### `[]byte`缓冲区相关的技巧

以下技巧在fasthttp中使用：

- 标准 Go 函数接收`nil`缓冲区

```go
var (
	// 两个缓冲区都是未初始化状态
	dst []byte
	src []byte
)
dst = append(dst, src...)  // is legal if dst is nil and/or src is nil
copy(dst, src)  // is legal if dst is nil and/or src is nil
(string(src) == "")  // is true if src is nil
(len(src) == 0)  // is true if src is nil
src = src[:0]  // works like a charm with nil src

// 这个 for 循环遍历切片 src 的每个元素，如果 src 为 nil，这个循环将不执行任何操作，直接跳过，不会引发程序错误。
for i, ch := range src {
	doSomething(i, ch)
}
```

因此，去除代码中对`[]byte`缓冲区的`nil`检查。例如：

```go
srcLen := 0
if src != nil {
	srcLen = len(src)
}
```

变为

```go
srcLen := len(src)
```

- 字符串可以使用`append`追加到`[]byte` buffer中

```go
dst = append(dst, "foobar"...)
```

- `[]byte`缓冲区可以扩展到其容量。

```go
buf := make([]byte, 100)
a := buf[:10]  // len(a) == 10, cap(a) == 100.
b := a[:100]  // is valid, since cap(a) == 100.
```

- 所有 fasthttp 函数都接受空的`[]byte`缓冲区

```go
statusCode, body, err := fasthttp.Get(nil, "http://google.com/")
uintBuf := fasthttp.AppendUint(nil, 1234)
```

- `string`和`[]byte` buffer可以在不进行内存分配的情况下进行转换。

```go
func b2s(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}

func s2b(s string) (b []byte) {
    bh := (*reflect.SliceHeader)(unsafe.Pointer(&b))
    sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
    bh.Data = sh.Data
    bh.Cap = sh.Len
    bh.Len = sh.Len
    return b
}
```

!!! 这是一种不安全的方式，结果的字符串和字节切片缓冲区共享相同的字节。“**确保在字符串仍然存在的情况下不要修改字节切片缓冲区中的字节！**”

### Good blog
fasthttp net/http对比 [fasthttp：比net/http快十倍的Go框架(server 篇)](https://www.luozhiyun.com/archives/574)
