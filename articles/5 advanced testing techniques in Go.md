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

在 interface 暴露给外部之前，要好好考虑。开发者通常倾向于暴露出接口，来给让用户模拟它们的行为。相反，你应该记录下你的接口体满足了什么接口，这样就不会在你的包和用户的包之间创建一个硬的依赖关系。 [error package](https://godoc.org/github.com/pkg/errors) 就是一个绝佳的例子。

