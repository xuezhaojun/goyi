# Self-referential functions and the design of options

[原文链接](Self-referential functions and the design of options)

我一直试着上下索求，一种在我写的Go包中，能正确的对选项类型做设置的途径。包会很复杂，一般程序的选项会有很多种。有不少的方法来做这件事，但是我要一种即使用方便，又不会要求太多API（无需用户太多心智负担）的方式，同时不会更着增加的需求膨胀。

我试过很多方法：选项结构体,大量的methods,变体构造函数等，并发现这些方法都没有那么好用。在过去一年的版本更替，以及和其他Gophers大量的沟通后，我终于找到了一种我喜欢的方式。你有可能会喜欢，也可能不喜欢，但是这确实是一种 自指函数（self-referential） 的有趣使用。

我希望我提起你的兴趣了。

让我们从一个简单的版本开始，我们将会改进这个版本，达到最终版本。

首先，我们定义一个 选项type,这是一个具有一个参数的方法，参数是Foo：

```go
type option func(*Foo)
```

关键点是，选项作为一个函数实现，我们可以调用这个这个函数来设置这个选项的状态，这可能看起来很怪，但是疯狂背后不无道理。

在给出option选项类型之后，我们下一步定义一个属于Foo的Option方法，可以将options作为参数传入。这个方法在同一个包种定义，即Foo定义的包：

```go
// Option sets the options specified.
func (f *Foo) Option(opts ...option) {
    for _, opt := range opts {
        opt(f)
    }
}
```

现在提供一个选项，我们定义一个方法看看，比如我们向设置Foo种的 verbosity这个字段。我们通过写一个还有明显名称同时返回一个option的方法，来提供了verbosity 选项，也就是闭包，在闭包内我们设置值：

```go
// Verbosity sets Foo's verbosity level to v.
func Verbosity(v int) option {
    return func(f *Foo) {
        f.verbosity = v
    }
}
```

为什么要返回一个闭包，而不直接的做设置呢？

因为我们不想让包的用户自己写闭包，同时也想让Option方法好用。

在调用的包内，我们可以通过以下方式设置值：

```go
foo.Option(pkg.Verbosity(3))
```

这样很简单，并且对于大部分的场景已经足够好了，凡是对于我写的那个包，我希望能够使用设置一些临时的量，也就是说希望Option方法能够返回上一个之前的状态。很简单，只要把之前的状态存到一个空的interface{}中即可，代码如下：

```go
type option func(*Foo) interface{}

// Verbosity sets Foo's verbosity level to v.
func Verbosity(v int) option {
    return func(f *Foo) interface{} {
        previous := f.verbosity
        f.verbosity = v
        return previous
    }
}

// Option sets the options specified.
// It returns the previous value of the last argument.
func (f *Foo) Option(opts ...option) (previous interface{}) {
    for _, opt := range opts {
        previous = opt(f)
    }
    return previous
}
```

调用方可以如之前一样的使用，并且调用方同时还可以保存上一个状态，只要保存第一次调用的结果即可。调用方代码如下：

```go
prevVerbosity := foo.Option(pkg.Verbosity(3))
foo.DoSomeDebugging()
foo.Option(pkg.Verbosity(prevVerbosity.(int)))
```

但是如你所见这里的类型断言实在是太恶心了，如果我们再稍微改进一下我们的设计，就会更好。

首先，重新定义一个option，是一个返回option类型，传入Foo的方法。

```go
type option func(f *Foo) option
```

这种自指函数定义很像状态机。现在我们使用的方法有一些不同：这是一个返回其相反的方法。

把Option的返回值设置为option类型，而不是 interface{} 类型：

```go
// Option sets the options specified.
// It returns an option to restore the last arg's previous value.
func (f *Foo) Option(opts ...option) (previous option) {
    for _, opt := range opts {
        previous = opt(f)
    }
    return previous
}
```

最后，对于option function的实现，其中的闭包部分现在需要返回一个option类型，而不是一个interface{},也就是说要返回一个自己undo的闭包，这也很简单，如下：

```go
// Verbosity sets Foo's verbosity level to v.
func Verbosity(v int) option {
    return func(f *Foo) option {
        previous := f.verbosity
        f.verbosity = v
        return Verbosity(previous)
    }
}
```

现在从客户端的视角来看，一切都那么美好：

```go
prevVerbosity := foo.Option(pkg.Verbosity(3))
foo.DoSomeDebugging()
foo.Option(prevVerbosity)
```

现在我们可以使用defer让客户端的代码更上一层楼：

```go
func DoSomethingVerbosely(foo *Foo, verbosity int) {
    // Could combine the next two lines,
    // with some loss of readability.
    prev := foo.Option(pkg.Verbosity(verbosity))
    defer foo.Option(prev)
    // ... do some stuff with foo under high verbosity.
}
```

值得注意的是，当 verbosity 的返回是一个闭包，而不是verbosity原先的值的时候，真实的原先值是隐藏的。如果你想知道这个值，你还需要魔改一下，不过目前已经够神奇了。

这些实现虽然看起来有些过头，不过真的去写也就每个option几行代码而已，并且通用性很高。最重要的是，这个东西对于包的用户来说真的是非常好。我对这个设计最终还是挺满意的。能使用Go的闭包来优雅的实现，再好不过了。