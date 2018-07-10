# 玩go六年，最佳实践 （部分）

[原文链接 - peter](https://peter.bourgon.org/go-best-practices-2016/)

> 部分内容并没有进行翻译，同时会注明原因



## 1. Development environment【未包含】

该部分是作者推荐的开发环境搭建，因为以下原因没有翻译：

* 已使用goland ide做开发，对于vim等其他工具不感兴趣
* 大部分人学习go第一件事就是搭建开发环境，无需阅读该部分内容

## 2. Repository structure 仓库结构

在我们等待项目成熟的时候，一些好的模式已经出现了。

世界上没有最好的仓库结构，但我认为确实存在一种对于很多项目都通用的模式。特别对，即提供二进制，又提供库的项目，集合非go代码资源的项目，非常有用。

基本概念为，设置两个最高级别的目录，**pkg 和 cmd**。在pkg下，创建你所用到的每一个库的目录。而在cmd下，创建你的每一个可执行文件的目录。你的所有go代码都应该位于这两个最高目录下面。

```
github.com/peterbourgon/foo/
  circle.yml
  Dockerfile
  cmd/
    foosrv/
      main.go
    foocli/
      main.go
  pkg/
    fs/
      fs.go
      fs_test.go
      mock.go
      mock_test.go
    merge/
      merge.go
      merge_test.go
    api/
      api.go
      api_test.go
```

你的所有东西仍然可以通过go get获取。路径可能会稍微长一些，但是命名方式却不会让人感到陌生。而且，你会有空间给一些非go的资源。比如，JavaScript 可以放在一个client 或者 ui 的子目录。Docker相关文件，或者其他帮助build的文件，可以存在于根目录，或者build子目录。Kubernetes mainfests这样的运行时配置可以存在于home子目录，等等。

> 关键：library文件放 pkg/ 子目录，可执行的文件放 cmd/ 子目录

当然，你应该使用完整的导入路径。就是说，在 cmd/foosrv 下的 main.go 文件，应该 `import "github.com/peterbourgon/foo/pkg/fs `。与此同时，要小心，包含 vendor 目录，对于你下游的使用者的影响。

> 关键：永远使用 **完整的** import 路径，永远！不要用相对路径

具有一点点结构，就可以让我们的仓库，拥有更包容的生态，和更好的易用性。

## 3. Formatting and style 风格

在这基本大家认知一致。这是go做的好的一面，我非常欣赏go社区的高度共识，以及这门语言的稳定性。

[Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments) 巨好,应该作为做code review标准时的最小标准集。然后，当在命名上有不同的意见时，最好的参考就是 Andrew Gerrand 的 [idiomatic naming conventions](https://talks.golang.org/2014/names.slide) 。

> 关键： 遵循 Andrew Gerrand 的 [idiomatic naming conventions](https://talks.golang.org/2014/names.slide) 

然后来说工具，工具现在越来越好了。你的编辑器应该包含 gofmt，有 goimports 那更好。go vet 工具（几乎）没有误报，所有你可以考虑将它作为你提交前的hook执行。同时可以考虑使用[gometalinter](https://github.com/alecthomas/gometalinter)  来检查 linting 问题，但是它会有误报，所有有时候你的 编辑你自己的linting配置 [encode your own conventions](https://github.com/weaveworks/mesh/blob/master/lint)  。

（安利goland，最新版是由自带 go-vet 的）

## 4. Configuration 配置

配置是运行时环境和程序中间的一层。配置需要 **非常明确，文档完善**，我仍然使用，并推荐 package flag，我也承认我希望它能浅显一些。我希望它由更标准，getopts 风格的长，短参数，我也希望它的usage文本能更紧凑些。

[12-factor apps](http://12factor.net/) 一文中，鼓励我们使用环境变量来作为配置，如果每一个var都定义为了flag，我觉得是可以的。明确性 非常重要：一个应用的运行时行为的改变，必须发生在可以被发现和记录的时候。

我在 2014 年说过，但是这里要再次强调：在你的main函数里定义和解析flags [define and parse your flags in func main](https://robots.thoughtbot.com/where-to-define-command-line-flags-in-go) 。只有main函数由权决定哪些flag将对用户可用。如果你的库代码要参数化，这些参数必须时构造函数的一个部分。将配置移动为全局变量，看似方便，实则不然：这么做会破环代码的模块化，对开发者和未来的维护者理解依赖关系造成困难，也让编写独立，并行的测试更加困难。

> 关键：只有main函数有权决定flags是否用户可用

如果要在社区发布一个用途广泛，包含以上特性的flag包，现在正是时候！如果已经有了，告诉我，一定用！

## 5. 程序设计

刚才，我就使用配置作为一个跳板，来稍微的讨论了一些程序设计的问题。作为开始，让我们先看一眼构造函数constructors。如果我们可以参数化我们所有的依赖，那我们构造函数会很大。

```go
foo, err := newFoo(
    *fooKey,
    bar,
    100 * time.Millisecond,
    nil,
)
if err != nil {
    log.Fatal(err)
}
defer foo.close()
```

有时候，这种构造函数会伴随一个config对象：包含了一个对象中，所有的 _可选_  参数的，参数结构体。我们假设 fooKey 是必要参数，其他的都是可选或者默认值具有意义。通过我会看到 config 对象，由碎片化的方式构造起来。

```go
// Don't do this.
cfg := fooConfig{}
cfg.Bar = bar
cfg.Period = 100 * time.Millisecond
cfg.Output = nil

foo, err := newFoo(*fooKey, cfg)
if err != nil {
    log.Fatal(err)
}
defer foo.close()
```

但是明显这么做更好：

```go
// This is better.
cfg := fooConfig{
    Bar:    bar,
    Period: 100 * time.Millisecond,
    Output: nil,
}

foo, err := newFoo(*fooKey, cfg)
if err != nil {
    log.Fatal(err)
}
defer foo.close()
```

这样对象就不会处于一些 中间状态，不可用状态。并且所有的字段都分隔，缩进良好，和fooConfig的定义一一对应。

注意 cfg 对应我们是即创建即用。在以下例子中，我们可以通过将结构体声明卸载newFoo的构造函数里面，再省一个中间状态，一行代码出来。

```go
// This is even better.
foo, err := newFoo(*fooKey, fooConfig{
    Bar:    bar,
    Period: 100 * time.Millisecond,
    Output: nil,
})
if err != nil {
    log.Fatal(err)
}
defer foo.close()
```

贼棒。

> 注意：用字面值初始化结构体，避免中间状态。结构体声明能内联就内联。

下一个主题是 让默认值有意义。注意观察可以发现，参数是有可能是nil的。为了讨论方便，假设是foo是一个io.Writer。当我们想使用它的时候，没有特殊的情况下，我们需要先检查它是否为空。

```go
func (f *foo) process() {
    if f.Output != nil {
        fmt.Fprintf(f.Output, "start\n")
    }
    // ...
}
```

很糟糕。如果能不检查，直接使用，那就更安全，也更方便了。

```go
func (f *foo) process() {
     fmt.Fprintf(f.Output, "start\n")
     // ...
}
```

所以我们要提供一个可用到默认值。作为一个接口类型，最好的方法是传一个接口的无操作实现。正好标准库ioutil就是一个无操作的 io.Writer,叫 ioutil.Discard。

> 注意: 通过 无操作 实现，来避免非空检查

我们可以将它传给 fooConfig对象，但是有可能存在调用者忘记的情况。放在构造函数中其实更加安全。

```go
func newFoo(..., cfg fooConfig) *foo {
    if cfg.Output == nil {
        cfg.Output = ioutil.Discard
    }
    // ...
}
```

这正是  *make the zero value useful* 的一个实践。我们允许零值（nil）产生运作正常的默认行为（no-op）。

> 注意：让零值有意义，特别是配置对象

回头来看看构造函数。参数 fooKey, bar,period,output 为全部的依赖了。一个foo对象要向运行成功，依赖于其中的每一个参数。如果要把我过去六年每天编写go代码和参与大型go项目学到的东西总结为一课，那就是：一定要让你的依赖清晰明确。

> 注意：参数依赖清晰明确！

一大堆维护负担，混乱，bug和技术债，我相信，都可以追溯到模糊不清的依赖关系上来。看看以下的方法：

```go
func (f *foo) process() {
    fmt.Fprintf(f.Output, "start\n")
    result := f.Bar.compute()
    log.Printf("bar: %v", result) // Whoops!
    // ...
}
```

fmt.Fprintf 方法是一个自足的，不影响也不受全局状态的影响，所以它是无依赖的。明显f.Bar是受依赖的。同时，有趣的是，log.Printf 是由一个包级别的全局 logger 对象提供，只是隐藏在了 Printf 方法之后，所以，它也是受依赖的。

遇到受依赖项怎么办？明确之。

因为 process 方法将 log 作为它的一部分，所以 process 方法和 foo 对象 都需要将一个 Logger 对象作为参数.比如：

```go
func (f *foo) process() {
    fmt.Fprintf(f.Output, "start\n")
    result := f.Bar.compute()
    f.Logger.Printf("bar: %v", result) // Better.
    // ...
}
```

我们会考虑一些特别的操作，比如写日志，只是偶尔的。也因此，我们会利用一些帮助程序，比如全局loggers，来减少一些代码量。但是日志，就像是仪表盘，往往对于一个服务来说是至关重要的。全局范围的隐藏依赖，不管是如logger一样简单，还是其他更加重要的，特定功能的，我们没有花功夫将其参数话的组件，都会让我们日后痛苦。

所以严格要求自己：将所有的依赖都明确！

> 注意： 日志也是受依赖的，就要其他组件，数据库句柄，命令行flag一样

当然我们也需要给我们的logger，添加一个有意义的默认值：

```go
func newFoo(..., cfg fooConfig) *foo {
    // ...
    if cfg.Logger == nil {
        cfg.Logger = log.New(ioutil.Discard, ...)
    }
    // ...
}
```

## 6. Logging and instrumentation

谈及这个问题，我真的有话想说: 我有大量的生成环境下的日志经验，这样很大程度的提高了我对于问题的尊重。日志的代价是高昂的，比你想的要高的多，而且很快会成为你系统的瓶颈。我在  [另一篇文章](https://peter.bourgon.org/blog/2016/02/07/logging-v-instrumentation.html) 中写的更加详细，但这里重新声明：

* 只打印具有操作价值的信息，因为这些信息是要被人或者机器阅读的
* 日志等级没必要分那么细 ---- 通常 info 和 debug 就够了
* 使用结构化的日 ---- 一点小私心，我推荐 [go-kit/log](https://github.com/go-kit/kit/tree/master/log) 
* Logger 是 受依赖的！