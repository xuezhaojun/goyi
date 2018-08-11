# Functional options for friendly APIs

[原文链接](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)

以下的文章都是我的个人观点，今天我在 dotGo 在做友好API的功能性选项。它们被编写的轻简可读。

我想先用一个故事开头。

2014年末，你的公司启动了一个革命性的，新的分布式社交网络，你的团队也非常机智的选择的Go作为产品开发的语言支持。

你的任务是编写关键的服务器组件。可能和如下类似：

```go
package gplusplus

import "net"

type Server struct {
    listener net.Listener
}

func (s *Server) Addr() net.Addr
func (s *Server) Shutdown()

// NewServer returns a nre Server listening on addr
func NewServer(addr string)(*Server ,error) {
    l,err := new.Listen("tcp",addr)
    if err != nil {
        return nil,err
    }
    srv := Server{listener:l}
    go srv.run()
    return &srv,nil
}
```

以上代码存在一些没有暴露出来的字段，需要初始化，同时还需要一个处理流入请求的goroutine。

这个包的API非常简单，非常易用。

但是出现了一个问题，在你发布了第一个版本之后，特性需求就不断的流入。

> Feature, features,features
>
> "慢客户端耗尽了我的资源"
>
> "你的服务器支持TLS吗"
>
> "我可以限制客户端的数量吗"
>
> "一些人一直尝试DOS我，我能限制某一些IP的连入吗？"
>
> 还有很多

很多的客户通常很慢才会回复，或者干脆就不会回复，你需要添加与这些慢客户端断开连接的支持。

你需要支持安全的连接。

然后，你获知某一额用户正在你的服务器上运行一个很小的vps，因此需要一个方法来限制同时连入的客户端数量

然后正对僵尸网络，你需要显示并发的连接数，进行速率的限制。

...还有很多很多。

现在，你需要将你的api的输入参数改为以下这样，才能满足所有的需求。

```go
一个提升了的API，大概吧
// NewServer return a new Server listening on addr.
// clientTimeout defines the maximum length of an idle
// connection,or forever if not provided.
// maxcons limits the number of concurrent connections.
// maxconcurrent limits the number of concurrent
// connections from a single IP address.
// cert is the the TLS certificate for the connection
func NewServer(addr string, clientTimeout time.Duration,maxconns, maxconrurrent int, cert *tls.Cert)
```

一个function不能很简单的展示出来，是一个不好的征兆。

举手看看，谁用过这样的API?

谁写过这样的API？

谁曾因为这样的代码导致程序崩溃过？

明显这样的解决方案非常的笨重，脆弱。并且不好理解。

你的package的新用户会对哪些参数是可选的，哪些是强制的，毫无头绪。

举个例子，如果我想创建一个用于测试的Server的实例,我是否需要提供一个TLS证书呢？如果不用，我要以什么作为这里的参数？

如果我不关系 maxconns,或者 maxconcurrent 我应该用什么参数呢？0值吗？0值听起来可以，但是基于我们实现的是什么特性，这个例子中可能会让你限制全部的并发连接数为0.

对于我来说，如果只要你能确保调用者能正确的使用以上的接口，那写api确实很容易。

这个例子中的方法是非常难用，难理解，我相信这个例子显示了一个现实中真实存在的问题。

现在我们来看看其他的，解决方式：

```go
Mang functions make light work
多点方法，生活更美好

NewServer(addr string)(*Server,error)
NewTLSServer(addr string,cert *tls.cert)(*Server,error)
NewServerWithTimeout(addr string,timeout time.Duration)(*Server,error)
NewTLSServerWithTimeout(addr string,cert *tls.Cert,timeout time.Duration)(*Server,error)
```

比起仅提供一个单一的方法，现在我们提供了很多的方法。

这种方法，当调用者需要一个secure的server的时候，他们会调用TLS的那一个。

如果他们需要一个链接的最大时间限制，他们可以选择哪个带有timeout时间参数的。

不幸的是，你也可以看出来，提供每一种可能的方法，真的太多了。

那么传入一个配置结构体如何？

```go
Passing a configuration struct
// A Config structure is used to configure the Server
type Config struct {
    Timeout time.Duration
    Cert *tls.Cert
}
func NewServer(addr string, config Config)(*Server,error)
```

使用config结构体是非常常见的方式。

使用这样的方案有以下好处:

* 配置信息可以随着时间的推移不断增加，但是创建Server的API接口并不会改变。
* 这样的方案，更容易写文档，之前在NewServer上面一大坨的注释，现在可以转化为一个文档化的结构体

但是这样的方案也不是完美的，它没有解决默认值问题，特别是0值有特定含义的时候。

```go
“我只是想建立一个server，不愿意想那么多！”

func NewServer(addr string, config Config)(*Server,error)

func main() {
    srv,_ := NewServer(addr,Config{}) // 默认端口是0，会报错
}
```

大多数的时间，你的API的用户都希望有默认值。

即使他们无意改变进行任何配置，调用的时候，还是得传入一些数值。

当他们读你样例代码的测试时，想要搞清楚怎么使用你的包，他们会看到一些神奇的空值，然后陷入无尽的困惑种。

对于我来说，这么做完全是错的。

我相信一个好的API不应该要求调用方为了极少的用例创建一些笨拙的数值。

我相信我们Go语言的程序员，应该努力让nil永远不作为任何public方法的参数。

如果我们需要传入配置信息，那它应该是尽可能自解释。

心里记着以上几点，我想说说我认为的好方法：

```go
变长的配置

func NewServer(addr string,config...Config)(*Server,error)

func main() {
    srv,_ := NewServer("localhost") // 默认的
    srv2,_ := NewServer("localhost",Config{
        Timeout:300 * time.Second，
        MaxConts:10,
    })
}
```

我们将`NerServer`改为接受复数的参数，可以解决一定要参数，却不常用的问题。

当你只想要默认值的时候，不再需要传入`nil`，或者其他的0值，调用默认的行为变为更加简洁的。

同时，现在`NewServer`只接受 config 作为参数，不再有pointer了。确保了调用者不能通过引用改变server内部的状态了。

我认为这样有了巨大的进步，但是如果我们偏执一些，还是会发现一些问题。

由于配置信息项是变长的，所有就得处理当用户传入多个配置的情况，虽然最多只有一个配置信息。有什么办法可以即使用变长的参数，同时提高配置参数的表现力呢？

我认为可以这么做

``` go
Function options

func NewServer(addr string,options ...func(*Server)) (*Server,error)

func main() {
    src,_:=NewServer("localhost")
    timeout := func(src *Server){
        srv.timeout = 60 * time.Second
    }
    tls := func(srv *Server){
        config := loadTLSConfig()
        srv.listener = tls.NewListener(srv.listener,&config)
    }
    srv2,_ := NewServer("localhost",timeout,tls)
}
```

在此，我要申明这种函数化选项的想法来源于Rob Pike的[Self referential functions and design](http://commandcenter.blogspot.com.au/2014/01/self-referential-functions-and-design.html) ，我建议每一个人都读一下。

与之前例子的区别是，配置不是保存于参数中，而是在一个可以操作`Server`的一个方法中。

```go
Functional options,cont

// NewServer returns a Server listening on addr
func NewServer(addr string,options ...fun(*Server)) (*Server,error){
    l,err := net.Listen("tcp",addr)
    if err != nil {
        return nil,err
    }
    
    srv := Server{listener:l}
    
    for _,option := range options {
        option(&srv)
    }
    
    return &srv,nil
}
```









