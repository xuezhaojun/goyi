# Error handling and Go

[原文链接 - Andrew Gerrand ](https://blog.golang.org/error-handling-and-go)

## Introduction

只要你写过go的代码，那你八成遇到过内建类型 error。Go的代码使用 `error` 值来表示一种不正常的状态。比如，`os.Open`方法会在它打开文件失败时，返回一个非空的（non-nil）`error`

```go
func Open(name string) (file *File, err error)
```

下面的代码使用os来打开一个文件，如果一个error出现，则会调用 `log.Fatal`打印错误信息，然后停止程序。

```go
f, err := os.Open("filename.ext")
if err != nil {
    log.Fatal(err)
}
// do something with the open *File f
```

你可以在仅仅知道这么一点，与error相关的知识的情况下，就用Go语言完成大量的事情。而是在这篇文章中，我们将会更加深入的看看error，并且讨论一些go语言中好的错误处理的实践。

## The err type

error 是一个 interface （接口）类型。一个 error 的变量，代表了任何可以将自己描述为一个字符串的变量。以下是接口的定义：

```go
type error interface {
    Error() string
}
```

error 类型，和所有的内建类型一样，是在  [universe block](https://golang.org/ref/spec#Blocks) 中 [prodeclared ](https://golang.org/ref/spec#Predeclared_identifiers)  (在宇宙块中预定义的，建议点开连接看一下这两个定义)

最常用到的 `error` 实现，就是在 [errors](https://golang.org/pkg/errors/) 这个包内，不对外暴露的 `errorString` 类型:

```go
// errorString is a trivial implementation of error.
type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}
```

你可以通过 `errors.New`方法来构造一个errorString的值。该方法会传入一个string，然后转化为 `errors.errorString` 然后返回一个`error`的值。

```go
// New returns an error that formats as the given text.
func New(text string) error {
    return &errorString{text}
}
```

下面是你可能看到的，使用 `errors.New` 的例子：

```go
func Sqrt(f float64) (float64, error) {
    if f < 0 {
        return 0, errors.New("math: square root of negative number")
    }
    // implementation
}
```

如果调用者传入一个负数的参数，那么就会收到一个非空的 error 值（实际就是一个 errors.errString 的值）。调用者可以通过调用 error 的 Error 方法，来获取对应的错误信息 ("math: square root of...")  ，或者直接打印它。

```go
f, err := Sqrt(-1)
if err != nil {
    fmt.Println(err)
}
```

fmt 包会通过调用 `Error() string` 方法，格式化一个error的值。

error是一个接口，而它的实现，则要对整合上下文中的相关信息负责。一个`os.Open`返回的error，会显示为 "open /etc/passwd:permission denied," 而不仅是 ‘’permission denied‘。目前我们上面写的`Sqrt`方法返回的error，就明显丢失了关于不合法输入的相关信息。

为了能添加这个信息，我们可以使用 `fmt` 包中一个非常有用的方法`Errorf`。这个方法会通过 `Printf`的规则，来格式化一个字符串，然后通过 `errors.New`方法创建一个`error`。

```go
if f < 0 {
    return 0, fmt.Errorf("math: square root of negative number %g", f)
}
```

在很多情况下，`fmt.Errorf`已经足够了，但是既然 `error`是一个接口，你就可以使用更加灵活的数据结构来实现`error`，让调用者获取到更多的关于错误的信息。

举个例子，假定一个调用者想查看传入`Sqrt`的错误参数。我们可以通过定义一个新的错误类型替代原来的`errors.errorString`来实现：

```go
type NegativeSqrtError float64

func (f NegativeSqrtError) Error() string {
    return fmt.Sprintf("math: square root of negative number %g", float64(f))
}
```

机智的调用者此时就可以通过[断言 type assertion](https://golang.org/ref/spec#Type_assertions)来检查是否是`NegativeSqrtError` 类型，然后特别的处理它，同时将新类型错误，传入 `fmt.Println` 或者 `log.Fatal` ，也不会有任何问题。

另一个例子，`json`包中，指定 `json.Decode`方法在解析json时，如果遇到一个语法错误，则会返回一个特殊的错误类型 `SyntaxError`。

```go
type SyntaxError struct {
    msg    string // description of error
    Offset int64  // error occurred after reading Offset bytes
}

func (e *SyntaxError) Error() string { return e.msg }
```

这个 `offset `字段并不是在默认的错误信息中输出，但是调用者使用它来为错误信息补充文件和行的信息。

```go
if err := dec.Decode(&val); err != nil {
    if serr, ok := err.(*json.SyntaxError); ok {
        line, col := findLine(f, serr.Offset)
        return fmt.Errorf("%s:%d:%d: %v", f.Name(), line, col, err)
    }
    return err
}
```

(这是来自 [Camlistore](https://perkeep.org/) 项目的 [真实代码](https://github.com/go4org/go4/blob/03efcb870d84809319ea509714dd6d19a1498483/jsonconfig/eval.go#L123-L135)的简化版)

`error`接口只要求实现`Error`方法即可；而特殊的error类型，则可能会有很多附带的方法。例如，[net](https://github.com/perkeep/perkeep%20perkeep.org) 包中的返回的error类型，除了通用方法，还有一些新增的方法:

```go
package net

type Error interface {
    error
    Timeout() bool   // Is the error a timeout?
    Temporary() bool // Is the error temporary?
}
```

客户端代码可以通过断言来探测一个`net.Error`，然后将 永久性网络错误 和 暂时性网络错误 区别开。例如，一个爬虫程序可以在遇到 暂时性问题 的时候，sleep 然后重试； 而当它遇到 永久性问题，就应该停止。

## 简化重复性错误处理

在Go中，错误处理非常重要。这个语言的设计与规则，就是鼓励你在遇到错误的时候，明确的检查它（其他语言中则是跑出异常然后catch他们）。在一些情况下，这会让go程序显得很臃肿，但是好在有很多方法可以让你用来简化重复的错误处理代码。

[App Engine](https://cloud.google.com/appengine/docs/go/) 这个应用中，它的 HTTP handler 会从数据库中检索一条记录，然后使用模板格式化它。

```go
func init() {
    http.HandleFunc("/view", viewRecord)
}

func viewRecord(w http.ResponseWriter, r *http.Request) {
    c := appengine.NewContext(r)
    key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
    record := new(Record)
    if err := datastore.Get(c, key, record); err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    if err := viewTemplate.Execute(w, record); err != nil {
        http.Error(w, err.Error(), 500)
    }
}
```

这个方法处理了，`database.Get`方法和`viewTemplate`的`Excute`方法返回的error。他们都只是将一个简单的错误信息带上 HTTP 的状态码 500（“Internal Server Error”） 然后传给用户。这看起来奏效了，但是一旦你要添加更多的HTTP handler，你就会发现重复写很多这样的错误处理代码。

为了不重复自己，我们定义了一个`appHandler`类型，它的返回值是 error 类型:

```go
type appHandler func(http.ResponseWriter, *http.Request) error
```

然后我们修改我们的`viewRecord`方法:

```go
func viewRecord(w http.ResponseWriter, r *http.Request) error {
    c := appengine.NewContext(r)
    key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
    record := new(Record)
    if err := datastore.Get(c, key, record); err != nil {
        return err
    }
    return viewTemplate.Execute(w, record)
}
```

这看起来比原版简单了，但是 http 包并不能识别一个返回 error 的 handler。所以我们还得让 `appHandler`实现

`http.Handler`接口的`ServeHttp`方法：

```go
func (fn appHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if err := fn(w, r); err != nil {
        http.Error(w, err.Error(), 500)
    }
}
```

`ServeHTTP`这个方法调用了 `appHandler` 这个方法，然后将返回的错误显示给了用户。注意这个方法的被调用方，`fn`,本身是一个函数。（Go 真厉害！）这个方法通过表达式`fn(w,r)`调用了函数本身。

现在，由于 `appHandler`是一个`http.Handler`，我们就可以通过http包中的`Handle`方法（而不是HandleFunc）来注册`viewRecord`了。

```go
func init() {
    http.Handle("/view", appHandler(viewRecord))
}
```

在这个基本的错误处理的基础上，我们可以让错误返回更加友好。比起直接返回错误信息，如果我们可以给用户一个简单的错误信息和http的状态码，同时将完整的错误信息日志记录到App Engine 开发者命令行中，用于检测问题，那就更好了。

首先，我们创建一个 `appError` 的结构体，包含了 `error`和一些其他的字段：

```go
type appError struct {
    Error   error
    Message string
    Code    int
}
```

然后我们修改 appHandler，让它返回 *appError :

``` go
type appHandler func(http.ResponseWriter, *http.Request) *appError
```

(通常返回一个特定类型的error会有问题，原因详见 [GO FAQ](https://golang.org/doc/faq#nil_error)，但是在这儿这么做没问题，因为 `ServeHTTP`是唯一一个可以看到value并且使用它的内容的位置)

然后我们让 `appHandler` 的 `ServeHTTP`方法,向用户显示`appError` 的`Message`和HTTP状态码，同时将全部的错误信息日志记录于开发者的console:

```go
func (fn appHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if e := fn(w, r); e != nil { // e is *appError, not os.Error.
        c := appengine.NewContext(r)
        c.Errorf("%v", e.Error)
        http.Error(w, e.Message, e.Code)
    }
}
```

最终，我们将完成了 `viewRecord`的升级，遇到错误时，它可以返回更多的信息了。

```go
func viewRecord(w http.ResponseWriter, r *http.Request) *appError {
    c := appengine.NewContext(r)
    key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
    record := new(Record)
    if err := datastore.Get(c, key, record); err != nil {
        return &appError{err, "Record not found", 404}
    }
    if err := viewTemplate.Execute(w, record); err != nil {
        return &appError{err, "Can't display record", 500}
    }
    return nil
}
```

这个版本的 `viewRecord`和原版一样的长，但是现在每一个行都有特定的意义，并且用户体验也提升了。

不要止步于此，我们还可以有更多的提高，比如：

* 给错误返回一个好看的http模板
* 当用户是管理员的时候，通过给HTTP Response写一个栈追踪，来让定位问题更简单
* 给 `appError` 写一个构造器来存储栈追踪信息
* 在 `appHandler`内恢复panic，打印日志为“Critical”，同时告诉用户“一个严重的问题正在发生”。这是避免让将复杂的编程问题暴露给用户的不错的尝试。可以通过查看[Defer,Panic,and Recover](https://golang.org/doc/articles/defer_panic_recover.html)了解更多。

## Conclusion

好错误处理，好软件！通过采用这篇文章描述的方法，希望你能写出更加可靠简洁的Go代码。