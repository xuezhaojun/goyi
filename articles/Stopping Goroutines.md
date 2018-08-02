# Stopping Goroutines

[原文链接](https://medium.com/@matryer/stopping-goroutines-golang-1bf28799c1cb)

> 想要再后台运行function，需要按三个键：g,o,space

以下是一个普通的函数调用：

``` go
DoSomething()
```

而以下，则是一个可以后台运行的调用：

```go
go DoSomething()
```

牛逼吧？

Go的另一个贼好的特性是Channels， goroutines之间可以通过Channel来通信。我们也可以利用channel来通知后台任务何时结束。

### Stop channels

通过channel，可以通知一个后台任务停止。

我们创建两个channel，一个 "stopchan" 和 一个 ''stoppedchan", 用于表示开始停止，和完成停止。

```go
// a channel to tell it to stop
stopchan := make(chan struct{})
// a channel to signal that it's stopped
stoppedchan := make(chan struct{})
go func(){ // work in background
  // close the stoppedchan when this func
  // exits
  defer close(stoppedchan)
  // TODO: do setup work
  defer func(){
    // TODO: do teardown work
  }()
  for { 
    select {
      default:
        // TODO: do a bit of the work
      case <-stopchan:
        // stop
        return
    }
  }
}()
```

这里我们使用关键词 “go” 来再后台运行一个函数。这个函数中启动了一个死循环，只有当 'select' 的 '<-stopchan' 发生的时候才会退出。同时，在 ‘default’ 下，执行工作。

