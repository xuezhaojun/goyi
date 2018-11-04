# How to Block Forever in Go

[原文链接](http://blog.sgmansfield.com/2016/06/how-to-block-forever-in-go/)

> 如何在Go中一直阻塞下去？

### Let me count the ways

让我算算有哪些方法，本文由部分代码变短和部分实战建议组成，下面所有的代码都非常简单的阻塞了当前的goroutine，同时不影响其他的goroutine。

### 为什么不用无限循环呢？

一个无限循环会占用100%的CPU，同时不允许Go runtime分配任何工作到这个核。而以下的所有的解决方案都同时满足永久阻塞和可以将处理器让给其他工作。

### 空select

这个方法将会尝试select返回的第一个选项，所以如果没有返回，它就会一直等下去

```go
func blockForever() {
    select{ }
}
```

### 自己等待自己

用过 `sync.WaitGroup`的都知道

```go
import "sync"

func blockForever() {
    wg := sync.WaitGroup{}
    wg.Add(1)
    wg.Wait()
}
```

### 双锁

一个`sync.Mutex`提供了对资源的保护，但是如果使用了两次呢？锁住！

```go
import "sync"

func blockForever() {
    m := sync.Mutex{}
    m.Lock()
    m.Lock()
}
```

### 读取空channel

空的，没有缓存的channel将会一直锁着，直到接受到东西

```go
func blockForever() {
    c := make(chan struct{})
    <-c
}
```

### 忙锁

这个方法其实没有锁住，它会将处理器让给其他工作，所以可用

```go
import "runtime"

func blockForever() {
    for {
        runtime.Gosched()
    }
}
```

### 自己握手

```go
func blockForever() {
    c := make(chan struct{}, 1)
    for {
        select {
        case <-c:
        case c <- struct{}{}:
        }
    }
}
```

### 睡美人锁

```go
import (
    "math"
    "time"
)

func blockForever() {
    <-time.After(time.Duration(math.MaxInt64))
}
```

### 这些有什么用？

第一种大概你永远不会用到。

举例，如果你有一个server，多个不同端口 http listener，你可以在你的`main()`中运行不同的goroutine, 然后用`select{}`永远锁住。这样当出现`panic()`的时候，程序就会退出。

如果出于一些原因，你并不想让服务在一个server failing的时候停止，你可以创建一个 `WaitGroup`然后用其等待所有的goroutine的完成。

其余的就单纯为了玩了，如果你有其他可以不独占CPU的方法，留言让我知道。