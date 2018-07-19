# 5 advanced testing techniques in Go

> Go 的五条进阶测试技巧

[原文链接](https://segment.com/blog/5-advanced-testing-techniques-in-go/)

Go 有着健壮的内置testing库。如果你写Go,你应该早就知道了。在这篇博文中，我们将会讨论一些可以提高你Go测试技巧的实用策略。根据我们自己在大型Go项目中的开发经验，这些策略可以有效的节约代码维护的时间。

### Use test suites

如果你看完这篇博文只能记得一件事，那我希望是： use test suites（使用测试集合）。suit testing  是 创建一个可以被不同实现所使用的，通用interface 的过程。如下文所展示，你可以看到我们传入多个不同的 `Thinger` 的实现，而他们都运行同样的测试。

```go
type Thinger interface {
    DoThing(input string) (Result, error)
}

// Suite tests all the functionality that Thingers should implement
func Suite(t *testing.T, impl Thinger) {
    res, _ := impl.DoThing("thing")
    if res != expected {
        t.Fail("unexpected result")
    }
}

// TestOne tests the first implementation of Thinger
func TestOne(t *testing.T) {
    one := one.NewOne()
    Suite(t, one)
}

// TestOne tests another implementation of Thinger
func TestTwo(t *testing.T) {
    two := two.NewTwo()
    Suite(t, two)
}
```

有些读者可能有过使用这种技术的工作经历。这种技术经常用于在 **基于插件的** 系统中，针对某一个interface编写的测试，将会用于检验，所有该interface的实现(implementations)，是否达到要求。

使用这个策略可能会帮你节约几个小时，几天的时间。并且，当你要替换两个底层系统的时候，如果应用这种方法，你无需编写许多额外的测试，并且心里很有谱。这种策略同时要求你需要创建一个interface，来定义你正在测试的外层表现。

这里由一个 [完整的例子](https://github.com/segmentio/testdemo/blob/master/suite/suite.go),虽然这里例子有点假，你可以想象其中一个实现是远程数据库，另一个是内存数据库

另一个贼好的例子是标准库中的 `golang.org/x/net/nettest` 包。它提供了验证一个net.Conn 是否满足其接口的方法。

### Avoid interface pollution

> 避免接口污染

说起Go中的测试，必将谈及interfaces。

在我们test的众多“武器”中，interface是最锋利的一把，所以如何正确的使用他们也就至关重要了。包中暴露出给用户使用的interface通常会导致两种情况：

a. 用户自己完成该interface实现的mock

b. 从包中导出interface的mock

> The bigger the interface, the weaker the abstraction.
>
> — Rob Pike, Go Proverbs

在 interface 暴露给外部之前，要好好考虑。开发者通常倾向于暴露出接口，来给让用户模拟它们的行为。相反，你应该记录下你的接口体满足了什么接口，这样就不会在你的包和用户的包之间创建一个硬的依赖关系。 [error package](https://godoc.org/github.com/pkg/errors) 就是一个绝佳的例子（error包并没有导出任何接口，error interface 是定义在error包外部的）。

当我们的程序中存在一个并不想导出的interface（一些公共包，但是又不想被外部引用）时，我们可以通过[internal/package subtree](https://golang.org/doc/go1.4#internalpackages) 的方式使得这个接口在包的范围之内。通过这种做法，我们不用再担心包用户可能依赖这个接口，这样一来，当出现新需求的时候，interface也可以灵活演变。我们一般为让者外部依赖创建接口，然后使用依赖注入，这样就可以再本地运行测试了。

这样包用户只需要将包的外层包装起来，实现一些他们自己的小接口来进行他们自己的测试。关于本章更多的内容，可以参考 [rakyll 关于接口污染的博文](https://rakyll.org/interface-pollution/)。

### Don't export concurrency primitives

> 不要导出并发原语

Go提供了相当简单的并发原语，这也导致了有时候它们会被过度使用。我们主要关心的时 channels 和 sync 包。有时候你会倾向于导出一个channel来给你的包用户，另外，还有一个常见的错误时再没有声明私有的情况下，内嵌了 sync.Mutex。这样做看起来无害，但是当你测试的时候，就会给你带来麻烦。

当你暴漏出 channels，你暴漏出的是包用户本不应该关系的复杂性。一旦channel从包中暴漏，测试这个channel消费的挑战也随即产生。包用户需要知道：

* channel上数据何时发送完
* 接受数据是否产生了某些错误
* 包是如何再完成后清理channel的
* 如何将包的api包装为一个接口，避免直接调用它们

考虑以下的一个读取队列的例子。这个例子中，库从队列中读取，然后暴露出一个channel让包用户读取。

```go
type Reader struct {...}
func (r *Reader) ReadChan() <-chan Msg {...}
```

现在你的用户向测试他的实现:

```go
func TestConsumer(t testing.T) {
    cons := &Consumer{
        r: libqueue.NewReader(),
    }
    for msg := range cons.r.ReadChan() {
        // Test thing.
    }
}
```

用户可能决定这里要使用依赖注入来写他们自己的messages

```go
func TestConsumer(t testing.T, q queueIface) {
    cons := &Consumer{
        r: q,
    }
    for msg := range cons.r.ReadChan() {
        // Test thing.
    }
}
```

但是 errors 要怎么处理？

```go
func TestConsumer(t testing.T, q queueIface) {
    cons := &Consumer{
        r: q,
    }
    for {
        select {
        case msg := <-cons.r.ReadChan():
            // Test thing.
        case err := <-cons.r.ErrChan():
            // What caused this again?
        }
    }
}
```

那么现在，我们要如何来模拟事件写入mock，来充分替代这个库的行为呢？是不是贼麻烦？

如果库中有同步API，那么我们就容易多了。

```go
func TestConsumer(t testing.T, q queueIface) {
    cons := &Consumer{
        r: q,
    }
    msg, err := cons.r.ReadMsg()
    // handle err, test thing
}
```

当你不确定的时候，记住往一个包里添加并发性容易，但是要从库中暴漏出来，那就几乎不能再移除了。最后，不忘了你的文档中说明你的struct或者package是否对多个goroutines的并发访问是安全的。

有时候，确实非得导出/暴露一个channel的情况下，限制它们为只读，或者只写吧。

### Use net/http/httptest

> 用 httptest

httptest包允许你无需准备一个server或者绑定一个端口的情况下，测试你的http.Handler。这会大大加速测试，并能让测试并行。

以下是两种测试方式，看起来不多，但是它其实帮你节约了不少的代码量和资源。

```go
func TestServe(t *testing.T) {
    // The method to use if you want to practice typing
    s := &http.Server{
        Handler: http.HandlerFunc(ServeHTTP),
    }
    // Pick port automatically for parallel tests and to avoid conflicts
    l, err := net.Listen("tcp", ":0")
    if err != nil {
        t.Fatal(err)
    }
    defer l.Close()
    go s.Serve(l)

    res, err := http.Get("http://" + l.Addr().String() + "/?sloths=arecool")
    if err != nil {
        log.Fatal(err)
    }
    greeting, err := ioutil.ReadAll(res.Body)
    res.Body.Close()
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(string(greeting))
}

func TestServeMemory(t *testing.T) {
    // Less verbose and more flexible way
    req := httptest.NewRequest("GET", "http://example.com/?sloths=arecool", nil)
    w := httptest.NewRecorder()

    ServeHTTP(w, req)
    greeting, err := ioutil.ReadAll(w.Body)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(string(greeting))
}
```

大概使用 httptest 的最大好处就是让你可以将你的测试按方法划分了，无需设置routers.middleware或者其他创建server，services，handler factories 时候需要的东西。

### Use a separate_test package

> 使用分离的_test包

大多数的测试，是创建于同一个包下的 pkg_test.go 文件中。分离测试包是指，你再一个独立的包中创建测试文件，而不是在要测试的包下面。比如在 foo_test 包下，而不是foo/ 下创建 foo_test.go。这样做有几个好处：可以解决测试中的循环依赖问题，可以避免脆弱测试，也让开发者可以体会使用自己包的感觉。如果你的包很难用，那么它估计也很难测试。

这种策略可以通过**限制对私有变量的访问**来防止脆弱测试。如果在分离测试包中测试失败，那么大概率使用你包的人在调用的时候也会失败。

这样的方法还有助于避免测试中的循环导入（import）。大部分包会依赖于你写的一些为测试的包，所以最终你会陷入一个场景：循环依赖自然而然的就发生了。这时需要有一个包能立于两个包之上。举个golang中的例子：`net/url`实现了一个URL解析器，而`net/http`中引用并使用了。但是`net/url`需要引入`net/http`来进行测试，如此，`net/url_test`诞生了。

如果你用一个分离的测试包，你可能会需要未导出（unexported ）的实例访问权限，可能这些实例之前还是可以访问到的。大多数人在测试基于时间的东西时，会第一次遇到这种情况。在这种情况下，我们可以使用一个附加的文件来仅在测试的情景下导出他们，_test.go 文件在编译的时候也会忽略掉。

### Something to remember

要记住，上面说的所有方法都不是银弹。最好的解决方法是永远谨慎的分析场景，然后决定最适合问题的解决方案。

想了解更多的Go测试技术吗？

看看以下博文：

* [Writing Table Driven Tests in Go](https://dave.cheney.net/2013/06/09/writing-table-driven-tests-in-go)  Dave Cheney
* [The Go Programming Language chapter on Testing.](http://www.gopl.io/) 

或者这些视频：

* [Hashimoto’s Advanced Testing With Go talk from Gophercon 2017](https://www.youtube.com/watch?v=yszygk1cpEc) 
* [Andrew Gerrand's Testing Techniques talk from 2014](https://talks.golang.org/2014/testing.slide#1) 