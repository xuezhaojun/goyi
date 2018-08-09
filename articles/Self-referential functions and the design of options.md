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

这样很简单，并且对于大部分的场景已经足够好了，凡是对于我写的那个包，我希望能