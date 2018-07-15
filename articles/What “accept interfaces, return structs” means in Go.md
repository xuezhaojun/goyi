# Go里面 “accpet interfaces, return structs” 是什么意思？

我在上一篇博文中提到一条通用准则 "accpet interfaces, return structs",同时也在和同事做code review的时候多次提到过，但是通过会被问到 “为什么？”。尤其在这条准则并不是一条刚性准则的前提下。这个思想的关键，以及怎么理解何时妥协它，就是要掌握如何在 **维持灵活性** 的同时，避免 **过早抽象** 的平衡之术。

## 过早抽象会导致系统复杂

> 计算机科学中的所有问题，都可以通过加中间层的方式解决，除了太多中间层这个问题
>
> --- *David J. Wheeler* 

软件工程师热爱抽象，个人来说，我从来没有见过任何一个，在写代码时，比创建一个东西的抽象的时，更投入的同事。接口会将Go中的结构体抽象出来，同事这种间接层也会存在嵌入的复杂性。遵循  [You aren’t gonna need it](http://c2.com/xp/YouArentGonnaNeedIt.html) （你其实不需要它）的软件设计哲学，在遇到这种需求之前，不要盲目的创造复杂性。一个从函数调用中返回接口普遍的理由是，希望用户专注于函数API。但是在Go中其实无需如此，因为接口是隐式的。API应该返回的是一个结构体。

> 永远只在你真正需要一个东西的再抽象它，而仅仅由于你预感到你需要它。

有些编程语言要求你预见你要使用的所有接口。而隐式接口的一大好处就在于，你可以在实现后再做抽象，而不是在实现前预先抽象。

## 旁观者清

你怎么决定什么时候需要一个接口呢？对于返回类型来讲，非常简单。你是写函数的那个人，所以你自己准确的知道什么时候你需要抽象返回值。你觉得需要抽象的时候就抽象它。

而对于函数的输入，控制权就不在你这里了。你可能会觉得你的数据库结构体已经足够了，但是用户可能还有把它用其他东西包起来。有一种很难，但不是不可能的方法，就是去预见每一个使用你方法的状态。这种在 可以控制输出，但是无法预料输入的不平衡，使得输入上的抽象，比输出的抽象，偏差更大。

## 清理无用的代码细节

简洁的另一个方面就是要清除非必要的细节。函数应该像食谱：输入一些东西，然后就可以得到一个蛋糕！没有食谱会包含他们不需要的东西。与之相同的是，函数也不应该将他们不需要的东西列出来。你觉得下列的函数怎么样？

```go
func addNumbers(a int, b int, s string) int {
  return a + b
}
```

很明显 s 并不需要，但是如果参数是结构体，那么这个问题就没那么明显了：

```go
type Database struct{ }
func (d *Database) AddUser(s string) {...}
func (d *Database) RemoveUser(s string) {...}
func NewUser(d *Database, firstName string, lastName string) {
  d.AddUser(firstName + lastName)
}
```

就像有太多配料的食谱一样，`NewUser` 将一个 Database 对象传入，携带了太多的东西。而它其实只需要一个 `AddUser`，但是却要传入诸如 `RemoveUser`的方法。接口可以让我们创建只依赖于我们需要的东西的函数。

```go
type DatabaseWriter interface {
  AddUser(string)
}
func NewUser(d DatabaseWriter, firstName string, lastName string) {
  d.AddUser(firstName + lastName)
}
```

Dave Cheney 在解释 [接口的分隔原则](https://en.wikipedia.org/wiki/Interface_segregation_principle) 时， [也写过这点](https://dave.cheney.net/2016/08/20/solid-go-design) ,他同时描述了一些其他的限制输入的好处。总结起来为如下：

> the results has simultaneously been a function which is the most specific in terms of its requirements–it only needs a thing that is writable–and the most general in its function （这句话是在原文中引用出来的，不结合上下文没法翻译，最好点上面的连接看下）

就像`addNumbers`中明显不需要s作为参数一样，`NewUser`同样是不需要一个可以删除用户的database对象的。

## 总结原因，理解意外

accpet interfaces,return structs 的主要原因如下：

* 移除不必要的抽象(后抽象原则)
* 用户对于函数输入的不确定性
* 简化输入

这些原因也让我们很好定义这些原则的意外情况。比如，如果你的方法确实可以返回多个类型的结构体，那就返回一个接口吧。同样的，如果函数是私有的，那么你就可以控制函数输入，也就是说不会造成过早的抽象。对于第三条原则，go是没有办法抽象结构体的成员变量抽象出来的，所有，如果你的方法需要结构体的成员（不仅仅是结构体上的方法），那么就必须传入一个结构体了。