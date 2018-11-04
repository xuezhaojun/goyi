# Error handling in Upspin

[原文链接](https://commandcenter.blogspot.com/2017/12/error-handling-in-upspin.html)

[Upspin](https://github.com/upspin/upspin)这个项目适用了一个自定义的包，[upspin.io/errors](https://godoc.org/upspin.io/errors) 来表示系统中出现的错误情况。这些errors满足标准Go error 接口，但是实现完全不同的自定义类型 [[upspin.io/errors.Error](https://godoc.org/upspin.io/errors#Error) ](https://godoc.org/upspin.io/errors#Error)，其中包含很多非常有用的特性。

我们将演示这个包如何工作，以及如何使用。承载着关于golang中如何进行error handle的大量讨论。

### Motivations 动机

这个项目进行了几个月之后，我们明确的发现，我们需要将代码中error的构建，表示，和处理 一致化。我们决定了实现一个自定义的errors包，并在某一个下午推出。虽然一些细节难免改变，但是这个包后面的核心思想一直延续下来了。它们即：

* 让创建 具有意义的error message更容易
* 让用于更容易理解错误
* 让error对程序员的调试有帮助

我们一边开发，一边还有新的动机加入，下面也会聊到

### A tour of package 包的旅程

upspin.io/errors 包会已errors的名字引入，也就是说它会替代Go‘s原生的errors包。

我们发现Upspin中有各种的错误信息：用户名，路径名，权限等等，所以这个包的起点就是，要能对不同的类型的错误进行构造，表示，报告。

这个包的核心即Error类型，一个Upspi Error的具体表示如下，它有很多字段，但是不是强制要设置的：

```go
type Error struct {
      Path upspin.PathName
      User upspin.UserName
      Op  Op
      Kind Kind
      Err error
}
```

Path ,User 字段表示受此次操作的路径和用户，注意，这两个都是string类型，但是在Upspin中含有独立的类型，是为了明确他们的用途 同时 允许类型系统捕获一些特定类型的标称错误

Op 字段表示当前的操作。这也是一个string类型, 表示了当前服务或方法的名字，比如“clinet.LookUp”,"dir/server.Glob"等

Kind 字段将错误按标准区分（Permission,IO,NotExit等）, 便于我们在error发生的时候，能看到大概的描述，并且提供了和其他系统交互的钩子。举个例子：[upspinfs](https://godoc.org/upspin.io/cmd/upspinfs) 