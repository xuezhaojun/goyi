# Using contexts to avoid leaking goroutines

> 使用context来避免goroutine泄漏

[原文连接](https://rakyll.org/leakingctx/)

 [context](https://godoc.org/pkg/context)  包让我们可以通过通过context的Done channel 来管理一个调用内的调用链。

在本文中，我们将会解释如何使用context包来避免goroutine泄漏。

假如，你有一个会内部启动一个goroutine的方法。一旦这个方法被调用，调用者将无法停止由这个方法启动的goroutine。

```go
// gen is a broken generator that will leak a goroutine.
func gen() <-chan int {
	ch := make(chan int)
	go func() {
		var n int
		for {
			ch <- n
			n++
		}
	}()
	return ch
}
```

以上的生成器会启动一个无限循环的goroutine，但是这个调用者会在n=5的时候就关闭了。

```go
// The call site of gen doesn't have a 
for n := range gen() {
    fmt.Println(n)
    if n == 5 {
        break
    }
}
```

一旦调用者完成了生成过程，这个goroutine就会一直执行下午，我们的代码就会产生一个goroutine泄漏。

我们可以通过通知goroutine内部，来避免这种情况，不过一般来说我们用户另一种更好的方法: 可取消的context。生成器可以select context的Done channel , 一旦context结束了，内部的goroutine也就取消了。

```go
// gen is a generator that can be cancellable by cancelling the ctx.
func gen(ctx context.Context) <-chan int {
	ch := make(chan int)
	go func() {
		var n int
		for {
			select {
			case <-ctx.Done():
				return // avoid leaking of this goroutine when ctx is done.
			case ch <- n:
				n++
			}
		}
	}()
	return ch
}
```

调用者的代码如下：

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // make sure all paths cancel the context to avoid context leak

for n := range gen(ctx) {
    fmt.Println(n)
    if n == 5 {
        cancel()
        break
    }
}

// ...
```

(canceled context很多文章都有提到，这篇文章中，我决定有意思的反而是这个gen()的写法，function是一等公民,chan也是呀)