# How to use go interface

[原文链接](https://blog.chewxy.com/2018/03/18/golang-interfaces/)

我偶尔会在我工作的时候做一些go的咨询和code review，因此，我看了不少别人的代码。虽然这是我个人的感觉，我发现一种 “Java风格”的接口使用方式，呈现上升状态。

这篇blog是基于我写go代码的经验，对于Go中使用interface的特别建议。

本文中的示例将会和两个包相关：`animal`和`circus`。我在这里写的也大多是一些处于包的边界的代码。

## Don't Do This

我经常能看到别人这么写：

```go
package animals 

type Animal interface {
	Speaks() string
}

// implementation of Animal
type Dog struct{}
func (a Dog) Speaks() string { return "woof" }
```

```go
package circus

import "animals"

func Perform(a animal.Animal) string { return a.Speaks() }
```

这就是我说的 “Java风格” 的接口使用方式。步骤大概是：

1. 定义一个接口
2. 第一个实现了接口的结构体
3. 定义要满足接口实现的方法

我总结为 “为了满足接口而写结构体”，这个[代码的味道](https://en.wikipedia.org/wiki/Code_smell)非常明显：

* 明显只有一个类型满足接口，同样没有明显的扩展方式
* 函数通常会采用具体类型，而不是接口类型

## Do This Instead

Go 的接口鼓励偷懒，而且是件好事。不再通过写一个类型来满足接口，应该写一个接口来满足需求。

我的意思是 - 不再于`animals`包中，定义一个 `Animal`接口，而是在 `circus`包中，根据使用点定义接口。

```go
package animals

type Dog struct{}
func (a Dog) Speaks() string { return "woof" }
```

```go
package circus

type Speaker interface {
	Speaks() string
}

func Perform(a Speaker) string { return a.Speaks() }
```

更加常见的做法是：

1. 定义类型
2. 根据一处用法来定义接口

这种方式下，简化了对于`animals`这个包中组件的依赖。简化依赖是你建立稳定软件的基础。

## Postel’s Law

编写好软件的一个优秀准则即是  [Postel’s Law](https://en.wikipedia.org/wiki/Robustness_principle) ，其中经常说到:

> “Be conservative with what you do, be liberal with you accept”   宽容的接受，保守的行动

用在go里即可以说是：

> “Accept interfaces, return structs”  接受信息，返回结构体

总的来说，这是把东西设计的稳定的好准则。go 中的主要代码单元是function。在设计function/method的时候，要遵循的方法如下：

```go
func funcName(a INTERFACETYPE) CONCRETETYPE
```

这里可以看到，我们接收任何实现了这个接口的东西 - 这个接口可以指任何接口，也可以是空接口。然后返回一个特定类型的值。当然 a 至少也得是空接口体。如同go箴言中说的：

```go
“the empty interface says nothing“ - Rob Pike
```

所以最好不要让functions接收 `interface{}`

### User Case: Mocking    用例:模拟

有一个非常好的，展示 Postel’s Law 在测试用例过程中的用法的例子。如果你有一个方法长得像这样：

```go
func Takes(db Database) error
```

如果 `Database`是一个接口，你就可以在测试的时候，提供一个`Database`的模拟实现，无需通过真实的 database 对象来进行测试。

## When Is It Acceptable To Define An Interface Upfront 什么时候可以提前定义接口

说真的，编程是一件相当自由的形式 - 没有真正意义上的硬性原则。你当然可以在前期就定义好接口。不会出现格式检查警察逮捕你的。在多个包的情况下，如果你已经知道自己的方法会调用一个包内的接口，那么就做吧。

太早定义接口 往往会散发 **过于工程** 这种code smell。但是却是存在你需要提前定义好接口的情况，我能想到几个：

* 密封的接口
* 抽象的数据类型
* 递归的接口

我依次解释一下：

### Sealed Interfaces 密封的接口

Sealed interfaces 只会在有多个包的情况下出现。Sealed interface是指没有对外暴露方法的接口。这意味着外部的使用者没有办法创建满足这个接口的类型。

如果定义是这样的:

```go
type Fooer interface {
	Foo() 
	sealed()
}
```

此时，只有定义了 `Fooer` 的包能创建一个满足Fooer的类型的值。Sealed interface主要用于让分析工具快速检查出非详尽模式匹配non-exhaustive pattern match ，比如通过 [sumtypes](https://github.com/BurntSushi/go-sumtype)  这个工具。

### Abstract Data Types 抽象的数据类型

另一个提前定义接口的用途是定义一个抽象的数据类型。可以是密封的，也可以是不密封的。

在`sort`包中，就有一个非常好的例子，它定义了一个可以被排序的集合：

```go
type Interface interface {
    // Len is the number of elements in the collection.
    Len() int
    // Less reports whether the element with
    // index i should sort before the element with index j.
    Less(i, j int) bool
    // Swap swaps the elements with indexes i and j.
    Swap(i, j int)
}
```

有很多人非常沮丧 - 排个序还得实现接口? 很多人都觉得多打这三行代码真麻烦。

但是我的观点是，这正是go中的优雅的泛型，应该受到鼓励。

### Recursive Interface

 这可能又是一种坏代码的味道，但是有时候确实会出现这种情况:

```go
type Fooer interface {
	Foo() Fooer
}
```

这种情况下就得先把接口定义清楚，一般的接口定义规范在这里就不适用了。

这种代码在创建可以操作的上下文的时候会用到。强依赖于上下文的代码，一般都会自己包含在一个包里，而是只有上下文会暴露出来，我确实也不常见这样的代码。

## 总结：

虽然本文有一章是以“Don't Do This”作为标题，但是目的并不是为了限制。而是希望大家都能思考一下边界条件 - 边缘情况就发生在这里。 

我个人发现 declare-at-point-of-use  这个部分极其实用。正因如此，我少了很多别人总遇到的问题。

我也会陷入写出 “Java风格”接口的情况，特别是刚写完python或者java后。设计过于工程和“一切都是类”的欲望在写完其他面向对象语言之后，再写go的时候，特别强烈。

所以这篇文章也算是自我提醒啦。

##### 昭君的总结：

* 根据一点用途来定义接口，而不是更具结构来定义接口 declare-at-point-of-use
* 传入接口，传出结构（宽入紧出）

