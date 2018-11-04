# Cancelling Multiple Goroutines（节选）

[原文地址](https://chilts.org/2017/06/12/cancelling-multiple-goroutines)

> 该文章前两个节比较简单，所以节选第三个章节之后的内容

### sync.WaitGroup

我们来试着整理和简化吧。这样做的原因是，如果我们要再添加一个goroutine，也有可能添加 10，20或者一百多个 - 那么要创建的 channels 真的多的让人头痛。

所以，让我们使用另一个 Go 提供的基础并发工具来替换 channels，那就是 [sync.WaitGroup](https://golang.org/pkg/sync/#WaitGroup) 。在下面的例子中，我们创建一个 WaitGroup (来代替两个channels) 然后使用它来通知所有的goroutines要结束了。记住，一旦我们创建了WatiGroup，千万不要copy它，我们需要传递它的引用。

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "sync"
    "time"
)

func main() {
    // a channel to tell `tick()` and `tock()` to stop
    stopChan := make(chan struct{})

    // a WaitGroup for the goroutines to tell us they've stopped
    wg := sync.WaitGroup{}

    // a channel for `tick()` to tell us they've stopped
    wg.Add(1)
    go tick(stopChan, &wg)

    // a channel for `tock()` to tell us they've stopped
    wg.Add(1)
    go tock(stopChan, &wg)

    // listen for C-c
    c := make(chan os.Signal, 1)
    signal.Notify(c, os.Interrupt)
    <-c
    fmt.Println("main: received C-c - shutting down")

    // tell the goroutine to stop
    fmt.Println("main: telling goroutines to stop")
    close(stopChan)
    // and wait for them both to reply back
    wg.Wait()
    fmt.Println("main: all goroutines have told us they've finished")
}

func tick(stop chan struct{}, wg *sync.WaitGroup) {
    // tell the caller we've stopped
    defer wg.Done()

    ticker := time.NewTicker(3 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case now := <-ticker.C:
            fmt.Printf("tick: tick %s\n", now.UTC().Format("20060102-150405.000000000"))
        case <-stop:
            fmt.Println("tick: caller has told us to stop")
            return
        }
    }
}

func tock(stop chan struct{}, wg *sync.WaitGroup) {
    // tell the caller we've stopped
    defer wg.Done()

    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case now := <-ticker.C:
            fmt.Printf("tock: tock %s\n", now.UTC().Format("20060102-150405.000000000"))
        case <-stop:
            fmt.Println("tock: caller has told us to stop")
            return
        }
    }
}
```

输出和上一版本的代码运行完全一致，所以我们改的没错。程序删了几行有加了几行，就让添加新的goroutine简单了不少：我们只需调用 wg.Add(1) 然后将 wg 和 stop 都作为参数传进去就行。输出如下：

```
$ go run 06/tidy.go 
tick: tick 20170612-221717.992723221
tock: tock 20170612-221719.992700713
tick: tick 20170612-221720.992722592
tick: tick 20170612-221723.992745407
^Cmain: received C-c - shutting down
main: telling goroutines to stop
tock: caller has told us to stop
tick: caller has told us to stop
main: all goroutines have told us they've finished
```

一帆风顺！但是，现在还有个问题即将出现。我们假设我们想要在一个goroutine中创建一个webserver。过去，我们会使用以下的代码创建一个。问题是server会阻塞goroutine直到server关闭。

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello, World!")
    })
    http.ListenAndServe(":8080", nil)
}
```

问题来了，我们要怎么通知 web server 停止呢？

### Context

在Go v1.7 中，[context](https://golang.org/pkg/context/) 包被引入，context包也是我们的下一个秘密武器。使用 Context  成为了过去几年中，Go中并发控制的瑞士军刀。

让我们快速浏览一下我们如何创建和关闭一个Context:

```go
  // create a context that we can cancel
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // pass this ctx to our goroutines - each of which would select on `<-ctx.Done()`
    go tick(ctx, ...)

    // sometime later ... in our case after a `C-c`
    cancel()
```

使用Context的一大好处在于，如果goroutine下又创建了goroutine，我们无需像之前一样创建更多的stop channels来通知子 goroutine 关闭，使用Context,每一个goroutine都会收到cancel。

在我们添加webServer之前，让我们用Context修改一下代码。要做的第一件事就是将 context 传入每一个 goroutine中。然后用 <- ctx.Done() 替换之前 select 的channel，保持在结束后通知 main()。

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "sync"
    "time"
)

func main() {
    // create a context that we can cancel
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // a WaitGroup for the goroutines to tell us they've stopped
    wg := sync.WaitGroup{}

    // a channel for `tick()` to tell us they've stopped
    wg.Add(1)
    go tick(ctx, &wg)

    // a channel for `tock()` to tell us they've stopped
    wg.Add(1)
    go tock(ctx, &wg)

    // listen for C-c
    c := make(chan os.Signal, 1)
    signal.Notify(c, os.Interrupt)
    <-c
    fmt.Println("main: received C-c - shutting down")

    // tell the goroutines to stop
    fmt.Println("main: telling goroutines to stop")
    cancel()

    // and wait for them both to reply back
    wg.Wait()
    fmt.Println("main: all goroutines have told us they've finished")
}

func tick(ctx context.Context, wg *sync.WaitGroup) {
    // tell the caller we've stopped
    defer wg.Done()

    ticker := time.NewTicker(3 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case now := <-ticker.C:
            fmt.Printf("tick: tick %s\n", now.UTC().Format("20060102-150405.000000000"))
        case <-ctx.Done():
            fmt.Println("tick: caller has told us to stop")
            return
        }
    }
}

func tock(ctx context.Context, wg *sync.WaitGroup) {
    // tell the caller we've stopped
    defer wg.Done()

    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case now := <-ticker.C:
            fmt.Printf("tock: tock %s\n", now.UTC().Format("20060102-150405.000000000"))
        case <-ctx.Done():
            fmt.Println("tock: caller has told us to stop")
            return
        }
    }
}	
```

和上一个版本又有一些小小的差别，但是现在我们可以：

1. 创建一个webserver，同时可以用Context的cancel关闭它
2. 给子goroutine传入同样的context，并通过该context实现关闭

然后，我们测试一下，输出一致，说明我们都做对了:

```
$ go run 07/tidy.go 
tick: tick 20170612-223954.341894561
tock: tock 20170612-223956.341886006
tick: tick 20170612-223957.341887182
tick: tick 20170612-224000.341927373
^Cmain: received C-c - shutting down
main: telling goroutines to stop
tock: caller has told us to stop
tick: caller has told us to stop
main: all goroutines have told us they've finished
```

现在是时候让我们的程序可以 serve Http 的请求了.

### The Webserver

在展示整个程序之前，我们来看一下webserver goroutine长什么样。这里妙在可以我们明确的通知它结束。

```go
func server(ctx context.Context, wg *sync.WaitGroup) {
    // tell the caller that we've stopped
    defer wg.Done()

    // create a new mux and handler
    mux := http.NewServeMux()
    mux.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Println("server: received request")
        time.Sleep(3 * time.Second)
        io.WriteString(w, "Finished!\n")
        fmt.Println("server: request finished")
    }))

    // create a server
    srv := &http.Server{Addr: ":8080", Handler: mux}

    go func() {
        // service connections
        if err := srv.ListenAndServe(); err != nil {
            fmt.Printf("Listen : %s\n", err)
        }
    }()

    <-ctx.Done()
    fmt.Println("server: caller has told us to stop")

    // shut down gracefully, but wait no longer than 5 seconds before halting
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // ignore error since it will be "Err shutting down server : context canceled"
    srv.Shutdown(shutdownCtx)

    fmt.Println("server gracefully stopped")
}
```

对于这个方法，我们只需要在main()中加两行即可：

```go
	// run `server` in it's own goroutine
    wg.Add(1)
    go server(ctx, &wg)
```

在这个程序的输出中，我将于第一次tick之后，通过 curl localhost:8080 发送一个请求到server，你会看到request开始，并在第二次tick前后完成，如以下：

```
$ go run 08/tidy.go 
tick: tick 20170612-230003.228960866
server: received request
tock: tock 20170612-230005.228893119
tick: tick 20170612-230006.228868513
server: request finished
tick: tick 20170612-230009.228863351
^Cmain: received C-c - shutting down
main: telling goroutines to stop
server: caller has told us to stop
tick: caller has told us to stop
server gracefully stopped
tock: caller has told us to stop
main: all goroutines have told us they've finished
```

 如我们所想，server也正确的关闭了。

再试一次，这一次我会在第二次tick时发送request，然后在第三次tick之前执行Ctrl-c，来演示server是如何优雅关闭的：

```
$ go run 08/tidy.go 
tick: tick 20170612-230408.026717601
tock: tock 20170612-230410.026710464
tick: tick 20170612-230411.026700385
server: received request
^Cmain: received C-c - shutting down
main: telling goroutines to stop
tick: caller has told us to stop
tock: caller has told us to stop
server: caller has told us to stop
Listen : http: Server closed
server: request finished
server gracefully stopped
main: all goroutines have told us they've finished
```

注意，tick()和tock()是先完成的，然后我们会有几秒钟的时间等待webserver完成它的request然后停止。在上一个例子中，server关闭的时候没有serveing任何request，所以 srv.ListenAndServe() 没有返回任何错误。在这个例子，server正在serving一个请求，并且会返回一个 `http:Server closed`错误，显示在 `request finished`信息上面，防止程序还在处理request，但是，它确实完成了，客户端会收到响应，一切都如预期的一样：

```
$ curl localhost:8080
Finished!
```

