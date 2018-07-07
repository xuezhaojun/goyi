# Effective error handling in Go

[原文链接 - Morsing‘s Blog](https://morsmachine.dk/error-handling)

## Introduction

go语言中最受争议的一件事就是错误的处理方式。以下有一些方案可以帮助你渡过大量错误处理的难关。

## Indented flow is for errors

在写go代码时，要：

```go
f, err := os.Open(path)
if err != nil {
    // handle error
}
// do stuff
```

不要：

```go
f, err := os.Open(path)
if err == nil {
    // do stuff
}
// handle error
```

这样，通过错误检查的例子读起来会是流畅的一列。

## Define your errors

处理错误的第一步是知道错误是什么？如果你的包有时会产生error，你的用户有可能对你产生的error感兴趣。你需要实现 error 接口，如下：

```go
type Error string

func (e Error) Error() string { return string(e) }
```

你的包的用户可以通过 **断言** 来判断是否是你的包产生的error：

```go
result, err := yourpackage.Foo()
if ype, ok := err.(yourpackage.Error); ok {
    // use ype to handle error
}
```

该方法同样可以将一些额外的信息导出给你的用户：

```go
type ParseError struct {
    File  *File
    Error string
}

func (oe *OpenError) Error() string {
    // format error string here
}

func ParseFiles(files []*File) error {
    for _, f := range files {
        err := f.parse()
        if err != nil {
            return &OpenError{
                File:  f,
                Error: err.Error(),
            }
        }
    }
}
```

如上，你的用户就可以明确的知道是哪一个文件解析失败了。

同时你需要警惕error包裹，当你包含一个error，有可能会造成信息的丢失：

```go
var c net.Conn
f, err := DownloadFile(c, path)
switch e := err.(type) {
default:
    // this will get executed if err == nil
case net.Error:
    // close connection, not valid anymore
    c.Close()
    return e
case error:
    // if err is non-nil
    return err
}
// do other things.
```

如果你包裹了一个 net.Error，如上的代码将不会看到网络失败了，并可能重复使用失效的链接。

首要原则是，如果你的包有用到其他的外部接口，保留调用他们时可能产生的错误。你的用户可能更关系这些错误。

### Errors as State

有时候你可能会持有一个error一段时间，1. 可以延迟报错，2. 你知道你可能会再次报错。

第一个情况比如是 bufio 包，当 bufio.Reader 遇到错误，不会马上报错，而是等buffer清空后才报错。

第二种情况是 go/loader包，当第一个参数报错时，会等待判断后续参数是否会报错。

## Use functions to avoid repetition

如果你有些重复的错误处理，你可以使用一个function统一处理

```go
func handleError(c net.Conn, err error) {
    // repeated error handling
}

func DoStuff(c net.Conn) error {
    f, err := downloadFile(c, path)
    if err != nil {
        handleError(c, err)
        return err
    }
    
    f, err := doOtherThing(c)
    if err != nil {
        handleError(c, err)
        return err
    }
}
```

修改后为：

```go
func handleError(c net.Conn, err error) {
    if err == nil {
        return
    }
    // repeated error handling
}

func DoStuff(c net.Conn) error {
    defer func() { handleError(c, err) }()
    f, err := downloadFile(c, path)
    if err != nil {
        return err
    }
    
    f, err := doOtherThing(c)
    if err != nil {
        return err
    }
}
```

以上

