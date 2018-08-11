# Your pprof is showing

> 显示你的pprof

[原文链接](https://mmcloughlin.com/posts/your-pprof-is-showing)

Golang的  [net/http/pprof](https://golang.org/pkg/net/http/pprof/)  真是超级好用：使用它可以细致的调式生产环境正在运行的服务器。正在进行的进程也是可以非常简单的将你的debugging信息导出的。在本文中，我们使用[zmap project](https://github.com/zmap) 来向你介绍现实中的实际问题，还有你可以采取的预防措施。

## Introduction

只要简单的 `import _ "net/http/pprof"`， 你就可以将终端的分析信息添加到一个HTTP服务器。

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	_ "net/http/pprof" // here be dragons
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello World!")
	})
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

服务器将不会简单的给你恢复"Hello World"给你，它同时会通过 /debug/pprof 给你汇报分析信息。

* /debug/pprof/profile:  30秒的CPU的分析
* /debug/pprof/heap: 堆分析
* /debug/pprof/goroutine?debug=1 :  所有的goroutine信息，带有栈跟踪信息
* /debug/pprof/trace:  做一次追踪

举个例子，假设我们启动了这个服务，并通过 [hey](https://github.com/rakyll/hey) 这个库模拟了一些请求，然后我们可以通过以下看到trace信息：

```go
$ wget -O trace.out http://localhost:8080/debug/pprof/trace
$ go tool trace trace.out
```

然后我们就可以看到服务器的细粒度信息了。这个特性在生产环境下追踪bug和性能问题的时候是无价的，但同时，能力越大，责任越大。

## Open Pprof Servers

这个方法简单的可怕。要求只有一个 import 即可！但是import则无处不在。它可能在你imported的一个库里。它让你可以自由的追逐goroutine泄漏，分析bug。但同时也引出了一个问题:

有多少的pprof server现在正在开放？

我们可以通过扫描IPv4来找到这个问题的答案。为了减少我们的搜索范围，我们可以限制到一些可能性较大的端口：

* 6060 一般是官方文档中建议的
* 8080一般是教程中建议的
* 80 标准的 http 端口
* 443 https的端口

首先，我得让你失望了：由于一些云服务提供商的威胁邮件，我最终没有完成搜索。但是我确信这是一个真实存在的问题。

使用[zmap project](https://github.com/zmap) ，可以将这些扫描简化到一行：

```go
$ zmap -p 6060 | zgrab --port 6060 --http="/debug/pprof/"
```

以下是我们的扫描结果：

* 至少有69个IP是将pprof放在6060端口
* 至少70个在8080端口
* 少数80端口，（没扫完，Google Cloud发邮件警告我不要发掘虚拟货币了）

虽然加密货币这件事很奇怪，但是我接受这个友情提示。我们发现有一些服务器把pprof暴漏出来了。我也打电话说了这个问题。

## Risks

安全问题永远是风险:

* Function 名字和文件路径将会显示
* 分析数据可能会暴露一些商业信息
* 分析将会降低性能，并给Dos攻击提供机会

也许对于你的应用来收，留一个debugging服务器看起来不会是很严重的安全漏洞，但是切记，千里之堤，溃于蚁穴。

## Prevention

Farsight Security  很早就预见并提出了一些非常好的建议：

> 一个简答有效的方式为，将pprof的http服务器，放在一个独立的localhost的端口，和应用的http服务器分开。

总的来说，就是要分两个服务器，简单的方式为：

* 应用服务放到 :80
* Pprof 服务则在 `localhost:6060`,且只能本地访问

这样做同时会限制你的服务器不能使用一些全局的http方法，你需要通过标准的方式来启动Pprof服务器了

```go
// Pprof server.
go func() {
	log.Fatal(http.ListenAndServe("localhost:8081", nil))
}()

// Application server.
mux := http.NewServeMux()
mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World!")
})
log.Fatal(http.ListenAndServe(":8080", mux))
```

如果处于一些原因，你要使用全局的 `http.DefaultServeMux` ，你可以做一个选择，然后如常执行。

```go
// Save pprof handlers first.
pprofMux := http.DefaultServeMux
http.DefaultServeMux = http.NewServeMux()

// Pprof server.
go func() {
	log.Fatal(http.ListenAndServe("localhost:8081", pprofMux))
}()

// Application server.
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World!")
})
log.Fatal(http.ListenAndServe(":8080", nil))
```

[professor package](https://github.com/mmcloughlin/professor) 这个包中，通过一些简单的方法达到了目的:

```go
// Pprof server.
professor.Launch("localhost:8081")

// Application server.
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World!")
})
log.Fatal(http.ListenAndServe(":8080", nil))
```

## Conclusion

net/http/pprof 牛逼！不要让别人看到你的debugging信息，听取以上的建议，就不会有问题