# Go Code Review

参考：

[Idiomatic Go](https://dmitri.shuralyov.com/idiomatic-go)

[CodeRiviewComments](https://github.com/golang/go/wiki/CodeReviewComments)

**本post旨在可以拿在手里快速进行code revice，非通篇翻译，对原文内容有选择删改**

## 简单问题

> 该部分如果有良好的编程习惯，则可以跳过

#### 特定词汇使用同一的拼写

Do

```go
// marshaling
// unmarshaling
// canceling
// cancelation
```

Don't do

```go
// marshalling
// unmarshalling
// cancelling
// cancellation
```

#### 句子间单空格

Do

```go
// Sentence one. Sentence two.
```

Don't do

```go
// Sentence one.  Sentence two.
```

#### Error变量的命名

Do

```go
// Package level exported error.
var ErrSomething = errors.New("something went wrong")

func main() {
	// Normally you call it just "err",
	result, err := doSomething()
	// and use err right away.

	// But if you want to give it a longer name, use "somethingError".
	var specificError error
	result, specificError = doSpecificThing()

	// ... use specificError later.
}
```

Don't do

```go
var ErrorSomething = errors.New("something went wrong")
var SomethingErr = errors.New("something went wrong")
var SomethingError = errors.New("something went wrong")

func main() {
	var specificErr error
	result, specificErr = doSpecificThing()

	var errSpecific error
	result, errSpecific = doSpecificThing()

	var errorSpecific error
	result, errorSpecific = doSpecificThing()
}
```

#### Error的推荐写法

Do

```go
if err != nil {
	// error handling
	return // or continue, etc.
}
// normal code
```

Don't

```go
if err != nil {
	// error handling
} else {
	// normal code
}
```

Also Do

```go
x, err := f()
if err != nil {
	// error handling
	return
}
// use x
```

Don't Do

```go
if x, err := f(); err != nil {
	// error handling
	return
} else {
	// use x
}
```

#### 人工写的注释后面要带一个空格

Do

```go
// This is a comment
// for humans.
```

Don't do

```go
//This is a comment
//for humans.
```

#### 注释要满足最基本的规则

以要注释的东西的名字开头，以句号结束。

```go
// Request represents a request to run a command.
type Request struct { ...

// Encode writes the JSON encoding of req to w.
func Encode(w io.Writer, req *Request) { ...
```

#### 文件命名使用使用单数来表示结合

Do

```go
github.com/golang/example/hello
github.com/golang/example/outyet
golang.org/x/mobile/example/basic
golang.org/x/mobile/example/flappy
golang.org/x/image/...
github.com/shurcooL/tictactoe/player/bad
github.com/shurcooL/tictactoe/player/random
```

Don't do

```go
github.com/golang/examples/hello
github.com/golang/examples/outyet
golang.org/x/mobile/examples/basic
golang.org/x/mobile/examples/flappy
golang.org/x/images/...
github.com/shurcooL/tictactoe/players/bad
github.com/shurcooL/tictactoe/players/random
```

#### 必须使用Gofmt

goland 下有自动的插件，每次保存文件时，可以自动运行Gofmt。

#### 错误信息

Do

```go
fmt.Errorf("something bad") // 无大写，无结尾符号
```

Don't do

```go
fmt.Errorf("Something bad。")
```



## 一般问题

#### Context

* 不要给结构体添加context，添加ctx参数到方法中
* 不要直接用context.Background()，除非你想不出来其他替代方案了
* 将ctx作为传入的第一个参数

#### Copying

* 如果有方法关联`*T`,不要copy`T`
* 当copy另一个包的结构体时，要小心。例如，当copy一个 Buffer，copy中的slice可能指向原本的array。

#### 声明空的Slice

Do

```go
var t []string
```

Don't do

```go
t := []string{}
```

第一种定义的是nil slice （json 中为 null）

第二种定义的是 non-nil ，zero-length的slice （json 中为 []）

[更多可见]([Understanding Nil](https://www.youtube.com/watch?v=ynoY2xz-F8s). )

#### 随机

使用 `crypto/rand `，不要使用`math/rand`

#### 样例

添加一个新包的时候，要包含可运行的Example，更多详见 [testable Example() functions](https://blog.golang.org/examples)

#### Goroutine 寿命

创建goroutine的第一件事，就是明确定义好它何时结束

#### 缩写名

Do

```
URLPony
ID
appID
ServeHTTP
```

Don't Do

```
UrlPony
Id
appId
ServeHttp
```

#### 接口

* 接口要放在 使用者所在的包，而不是实现者所在的包
* 先使用，后定义接口

#### Named Result Parameters

Do

```go
func (n *Node) Parent1() *Node
func (n *Node) Parent2() (*Node, error)
```

Don't do

```go
func (n *Node) Parent1() (node *Node)
func (n *Node) Parent2() (node *Node, err error)
```

Also Do

```go
// Location returns f's latitude and longitude.
// Negative values mean south and west, respectively.
func (f *Foo) Location() (lat, long float64, err error)
```

Don't 

```go
func (f *Foo) Location() (float64, float64, error)
```

#### 接受 Receiver

* 更推荐 Pointer 类型
* 命令统一且短(client as cl)