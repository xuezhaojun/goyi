我希望能够使用设置一些临时的量，也就是说希望Option方法能够返回上一个之前的状态。很简单，只要把之前的状态存到一个空的interface{}中即可，代码如下：

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