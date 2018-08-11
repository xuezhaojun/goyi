# Don’t use Go’s default HTTP client (in production)

> 不要再生成环境下使用Go默认的HTTP 客户端

[原文链接](https://medium.com/@nate510/don-t-use-go-s-default-http-client-4804cb19f779)

通过Go来写访问HTTP服务的代码十分的简单有趣，我写过很多API客户端的包，并认为这是一项非常有意思的任务。

但是其实，我也曾有过陷入陷阱的经历，它会崩溃掉你的程序：HTTP 默认的客户端。

---

Go 的 http 包，默认不会指定请求的超时，允许服务一直占用你的goroutine，当你要链接外部的服务时，永远记得，自定义一个 http.Client！

---

### 问题示例

假设你想要通过 spacely-sprockets.com 漂亮的JSON REST API 来访问  spacely-sprockets.com  网站，在 Go中，你可能是这么写的:

```go
// error checking omitted for brevity
var sprockets SprocketsResponse
response, _ := http.Get("spacely-sprockets.com/api/sprockets")
buf, _ := ioutil.ReadAll(response.Body)
json.Unmarshal(buf, &sprockets)
```

你写好了代码（当然error 处理请补上），编译，运行，一切看起来都正常，现在，你API打包，然后作为web应用的一部分加入。你的应用中的一个页面会通过调用这个API显示Spacely Sprockets 库存的列表。

一切看起来都没有问题，直到有一天你的app突然不响应了。你查看日志，但是没什么对于定位问题有益的帮助。你检查你的监控工具，但是CPU,内存，I/O都看起来很有嫌疑，你将它沙盒运行进行检查，但是没什么问题，究竟怎么了？

失望之中，你刷了刷Twitter，发现Spacely Sprockets  的开发团队说他们刚停电了，但是目前已经恢复了。你检查了他们的API状态也买你，发现他们的中断就在你的中断的几分钟之前。一切看起来这么的巧合，但是你也不能确定有何相关，因为你的API代码优雅的处理的error。要知道为啥出问题，你还离得很远。

---

### The Go HTTP package

Go 的HTTP包使用了一个叫作Client的结构体来管理内部的连接。Clinent 是并发安全的结构体，包括配置信息，管理TCP状态，处理cookies 等，当你使用 `http.Get(url)`,你就是在用 `http.DefaultClient`,一个拥有默认配置的包级别变量，定义如下：

```go
var DefaultClient = &Client{}
```

除此之外，http.Client 有一个配置 timeout, 可以一些检查长连接状态。这个值默认是0，表示无超时。对于这个包来说，这个默认值是有意义的，但是对于我们的服务来说，就会导致上面的问题。结果表明， Spacely Sprockets’ API  停止，会导致连接被挂起。它们会一直被挂着，只要故障的服务器决定等待。因为API调用服务于我们的用户请求，这导致服务用户请求的goroutine也被挂起来了。一旦点击 sprockets 页面的人足够多，app就会因为资源不足而挂掉。

以下是为了演示问题的一个Go程序：

```go
package main
import (
  “fmt”
  “net/http”
  “net/http/httptest”
  “time”
)
func main() {
  svr := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    time.Sleep(time.Hour)
  }))
  defer svr.Close()
  fmt.Println(“making request”)
  http.Get(svr.URL)
  fmt.Println(“finished request”)
}
```

这个服务一旦运行，会程序会发出一个请求到server，然后会sleep一个小时，也就导致，程序会等待一个小时然后退出。

---

### 解决方案

解决方案是，永远，记住是永远的在你使用的时候，定义一个有timeout的 http.Client, 以下是一个例子：

```go
var netClient = &http.Client{
  Timeout: time.Second * 10,
}
response, _ := netClient.Get(url)
```

这样终端发送请求会有10秒的超时限制，如果API的Server超过了这个时间，Get()将会返回如下的error：

```go
&httpError{
  err:     err.Error() + " (Client.Timeout exceeded while awaiting headers)",
  timeout: true,
}
```

如果你需要更加细粒度的，对request生命周期的控制，你可以额外的指定一个定制化的 net.Transport 和一个 net.Dialer。Transport 是用于客户端管理底层tcp连接的结构体，而Dialer是管理连接建立的结构体。Go的net包有一个默认的Transport和Dialer，而以下则是定制的：

```go
var netTransport = &http.Transport{
  Dial: (&net.Dialer{
    Timeout: 5 * time.Second,
  }).Dial,
  TLSHandshakeTimeout: 5 * time.Second,
}
var netClient = &http.Client{
  Timeout: time.Second * 10,
  Transport: netTransport,
}
response, _ := netClient.Get(url)
```

这一段代码会限制TCP连接和TLS握手的超时时间，同时也会建立一个端对端的请求超时时间。还有其他的一些选项比如keep-alive超时，如果你需要的话，可以试试。

---

### 总结

Go的net和http包都是精心设计，方便易用的HTTP通信基础包。不过，对于默认timeout的缺失，会导致在做请求的时候出现问题。一些其他的语言，比如Java也有同样的问题，而一些(比如Ruby有一个60秒的默认读超时)则没有。不设置超时会让你的应用收到正在连接的那个远程服务的影响，一个故障的服务会挂住你的连接，挂死你的应用。