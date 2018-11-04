# Errors are values

[原文链接 - Rob Pike](https://blog.golang.org/errors-are-values)

go 程序员，特别是刚接触 go 语言的go程序员，都会讨论到一个点 ---- 如何处理 errors。 这些讨论往往会转化为对以下代码出现次数的哀叹：

```go
if err != nil {
    return err
}
```

 我们最近扫描了所有我们能找到的开源项目，并且发现这个代码一两页才出现一次，比大家想的要少的多。但是，如果还是有人坚持认为，他要敲很多的``` if err != nil```的代码。那么一定是哪里出现了问题，而且很明显就是go本身。

好在这个问题很好纠正。事情可能是这样的：一个go语言的新人提出问题 “要怎么处理error呢？”，学会之后，就停在那里了。其他语言中，可能会需要 try-catch 或者其他机制来处理error。因此，程序员会想，之前什么时候用到 try-catch ,现在我go中就敲 if err != nil。慢慢的代码中就多出了很多这样的片段，结果显得笨笨的。

先不管这种解释是否合适，这些程序员没有一个关于errors的基本认知：errors are values，错误也是值。

可以对values编程，errors是values，那么你也可以对errors进行编程。

当然最常见的情况就是判断error的值是不是nil，但是其实你可以用error来做数不尽的事，稍加应用就可以明显改善程序，并消灭大部分死板的错误检查代码。

这里有一个bufio包中Scanner例子，它的Scan方法执行了底层IO，会导致一个error的产生，然而scan方法并没有返回一个error，而是返回了一个布尔值，和另一个在scan结束后调用的，分离的方法，来报告是否又error产生。客户端的代码如下：

```go
scanner := bufio.NewScanner(input)
for scanner.Scan() {
    token := scanner.Text()
    // process token
}
if err := scanner.Err(); err != nil {
    // process the error
}
```

这里有一处对于error的检查，但是它仅出现并执行一次！如果这个 scan 方法如下定义：

```go
func (s *Scanner) Scan() (token []byte, error)
```

那么上面的那个例子就是变成下面这样：

```go
scanner := bufio.NewScanner(input)
for {
    token, err := scanner.Scan()
    if err != nil {
        return err // or maybe break
    }
    // process token
}
```

这虽然不难，但是却有一个非常重要的区别。在这个代码中，客户端必须每次循环都进行一次错误检查，但是在真正的Scanner API，错误处理是由一些关键的API中抽象出来的。使用真实的API，客户代码更加的自然：不断循环一致到结束，然后再考虑错误处理的问题，没有让错误处理影响控制流。

这一切是怎么发生的呢？当Scan遇到I/O error的时候，就会记录并返回false。另一个分离的方法 Err 会再客户端问询的时候反馈这个err。虽然看起来很微不足道，但是这样就和，到处敲```if err != nil```或者要求客户端每次获取token后都检擦错误，不一样了。

值得强调的是，不论设计如何，如果由error还是要检查一定要检查的。我们讨论的不是如何避免错误检查，而是如何应用这门语言优雅的错误检查。

重复的错误检查代码的议题，是在我参见2014年东京的GoCon大会出现的，另一个经验丰富的gopher也在他的twitter（用户名 @jxch_）中抱怨错误处理，他的示例代码如下:

```go
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}
_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}
_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}
// and so on
```

代码重复性很高，在实际代码中更长。在实际代码中可以使用一个帮助方法来解决这个问题，虽然也不那么容易，但是在这种理想情况下，如下的方法可以有所帮助：

```go
var err error
write := func(buf []byte){
    if err != nil {
        return 
    }
    _,err = w.Write(buf)
}
write(p0[a:b])
write(p1[c:d])
write(p2[e:f])
if err != nil {
    return err
}
```

我们可以通过借鉴之前Scan的方法，让代码更清晰，通用性更高。我之前在讨论中提到过这种技术，但是@jxch_并没有使用它。在语言阻碍造成的长时间沟通下，我询问是否我可以借用他的笔记本电脑，通过写一些代码来给他演示。

我定义了一个叫做 errWriter 的对象，如下：

```go
type errWriter struct {
    w   io.Writer
    err error
}
```

然后给它定义了一个方法 write（小写，为了和Write方法区分），这个write方法调用了Write方法，然后记录了err用作未来的引用。

```go
func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return 
    }
    _,ew.err = ew.w.Write(buf)
} 
```

一旦错误出现，write方法将直接返回，但是err会保存下来。

有了errWriter和它的write方法，上面的代码可以重构如下：

```go
ew := &errWriter{w: fd}
ew.write(p0[a:b])
ew.write(p1[c:d])
ew.write(p2[e:f])
// and so on
if ew.err != nil {
    return ew.err
}
```

现在甚至比之前使用闭包还要清晰，也让代码流程种的write操作在页面上更可见。这下再没有笨笨的感觉了。使用error作为值编程让代码更棒了。

可能同一包中其他地方的代码也可以使用这种思想，甚至直接使用 errWriter。

同时，当 errWriter 存在，那么它可以做更多的事。特别是一些更实用的例子中，比如累计字节数，可以将合并写入到同一个buffer中，然后自动传送，等等。

实际上，这种模式也经常出现在标准库中，比如 [archive/zip](https://golang.org/pkg/archive/zip/) 和 [net/http](https://golang.org/pkg/net/http/) 中。bufio package's Writer 就是完全实现了 errWriter 的思想。虽然[bufio.Writer.Write](https://golang.org/pkg/bufio/)会返回一个err，但是只是为了实现[ io.Writer](https://golang.org/pkg/io/#Writer) 的接口。bufio.Writer的Write方法和上面errWriter的表现一模一样，通过 Flush 方法来报错，所以我们的例子也可以这么写：

```go
b := bufio.NewWriter(fd)
b.Write(p0[a:b])
b.Write(p1[c:d])
b.Write(p2[e:f])
// and so on
if b.Flush() != nil {
    return b.Flush()
}
```

这种方案的通病则是：再error出现之前，没法判断程序的完成度。如果这个信息很重要，那么则需要一个更加细粒度的方案，通常在末尾放一个 all or nothing 的错误检查就足够了。

我们现在只探讨了避免重复错误检查的一种手段。铭记errWriter这种方法并不是唯一的方法，并且这种方案也并不是适用于所有情况。关键在于，errors are values ，而go语言的强大之处就是你可以对其进行编程。

使用这门语言来简化你的错误处理吧。

但是记住，不管怎么做，一定要做错误检查！

最后，我和@jxck_交流的全部故事，包括他录制的一个小视频，可以在[他的blog](http://jxck.hatenablog.com/entry/golang-error-handling-lesson-by-rob-pike)看到





















































