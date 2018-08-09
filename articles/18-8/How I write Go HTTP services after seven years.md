# How I write Go HTTP services after seven years

[原文链接](https://medium.com/statuscode/how-i-write-go-http-services-after-seven-years-37c208122831)

我已经从Go r59（ 一个1.0release之前的版本）之前就开始写Go了，而且在过去的7年一直都又建立HTTP API 和 服务。

在  [Machine Box](https://machinebox.io/?utm_source=matblog-3May2018&utm_medium=matblog-3May2018&utm_campaign=matblog-3May2018&utm_term=matblog-3May2018&utm_content=matblog-3May2018)  工作时，我大部分的工作就是创建各式各样的APIs。机器学习对于大部分开发者来说即遥远又发杂，所以我的工作就是通过API简化之，到目前为止，我的工作得到的反馈还是很好。

> 如果你还没试过Machine Box，请试试，然后告诉我们你觉得怎么样

我写服务的方式在这几年中，一直在改变，所以我想分享现在，今天，我时怎么写服务的，希望对你的工作有参考价值。

### A server struct

我所有的组件都有一个`server`结构体，一般都是长这样：

```go
type server struct {
    db     *someDatabase
    router *someRouter
    email  EmailSender
}
```

* 结构体的字段都是需要分享的依赖

### routes.go

我在每一个组件中都有一个`routes.go`文件，所有的路由都在这里：

```go
package app
func (s *server) routes() {
    s.router.HandleFunc("/api/", s.handleAPI())
    s.router.HandleFunc("/about", s.handleAbout())
    s.router.HandleFunc("/", s.handleIndex())
}
```

这样写非常方便，因为大部分的代码维护工作都是从URL和一个error报告开始 - 只要看一眼`route.go`文件就知道我们要找哪里的问题

### Handlers hang off the server

我的HTTP handlers时挂在server下的

```go
func (s *server) handleSomething() http.HandlerFunc { ... }
```

这样Handlers可以通过 s 这个server变量访问到各种依赖

### Return the handler

我的handler 犯法并不是真的 handle 请求，它们返回一个处理请求的方法。

这样给我们一个闭包的环境，可以让我们的handler被这样操作:

```go
func (s *server) handleSomething() http.HandlerFunc {
    thing := prepareThing()
    return func(w http.ResponseWriter, r *http.Request) {
        // use thing        
    }
}
```

这里的 `prepareThing` 只被调用了一次，所以你做一次prepareThing，然后就可以在每一个handler中都使用thing了。

当然这里你要保证这个data是只读的，如果handlers会写入东西，记得要加mutex一类的保护措施。

### Take arguments for handler-specific dependencies

如果有特别的handler存在依赖，将其作为一个参数

```go
func (s *server) handleGreeting(format string) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, format, "World")
    }
}
```

`format`变量对于handlers是可操作的。

### HandlerFunc over Handler

我几乎所有的情况都会使用 `http.HandleFunc`，而不是`http.Handler`

```go
func (s *server) handleSomething() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ...
    }
}
```

两者都差不多，所以从其中挑一个更易读，更易理解的。对于我来说 ，就是 `http.HandleFunc`

### Middleware are just Go functions

中间件只是一个Go的function，将一个`http.HandleFunc`作为参数，并返回一个新的`http.HandleFunc`。中间件能在调用原先的handler之前或/和之后被调用 - 甚至中间件可以决定是否调用原本的handler。

```go
func (s *server) adminOnly(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if !currentUser(r).IsAdmin {
            http.NotFound(w, r)
            return
        }
        h(w, r)
    }
}
```

这段handler中的逻辑可以决定是否调用原本的handler。

一般我会将中间件列在`routes.go`文件中：

```go
package app
func (s *server) routes() {
    s.router.HandleFunc("/api/", s.handleAPI())
    s.router.HandleFunc("/about", s.handleAbout())
    s.router.HandleFunc("/", s.handleIndex())
    s.router.HandleFunc("/admin", s.adminOnly(s.handleAdminIndex))
}
```

### Request and response types can go in there too

如果一个endpoint有它自己的请求和响应类型，一般就是说它们只对特定的handler有效。

如果是这种情况，你可以在function内部定义他们。

```go
func (s *server) handleSomething() http.HandlerFunc {
    type request struct {
        Name string
    }
    type response struct {
        Greeting string `json:"greeting"`
    }
    return func(w http.ResponseWriter, r *http.Request) {
        ...
    }
}
```

这样可以让你命名这些类型，而不用根据特定的handler版本来考虑。

在测试代码中，你可以copy这些类型到你的test方法中，或者...

### Test types can help frame the test

如果你的request/response隐藏在handler中，你可以在你的测试代码中声明新的类型

这里有机会为将来需要理解你代码的人，做一些铺垫。

比如，我们在代码中有一个`Person`类型，我们在很多的endpoints都使用到了它，如果我们一个`/greed`endpoint,我们可能只关系他们的名字，所以我们可以这样写测例：

```go
func TestGreet(t *testing.T) {
    is := is.New(t)
    p := struct {
        Name string `json:"name"`
    }{
        Name: "Mat Ryer",
    }
    var buf bytes.Buffer
    err := json.NewEncoder(&buf).Encode(p)
    is.NoErr(err) // json.NewEncoder
    req, err := http.NewRequest(http.MethodPost, "/greet", &buf)
    is.NoErr(err)
    //... more test code here
```

这样测试就更清晰了，我们只需要关系person的`Name`字段即可。

### sync.Once to setup dependencies

如果我在handler 准备阶段做了哪些昂贵的操作，我会将它延迟到handler第一次被调用的时候，这会让application启动的更快

```go
func (s *server) handleTemplate(files string...) http.HandlerFunc {
    var (
        init sync.Once
        tpl  *template.Template
        err  error
    )
    return func(w http.ResponseWriter, r *http.Request) {
        init.Do(func(){
            tpl, err = template.ParseFiles(files...)
        })
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        // use tpl
    }
}
```

`sync.Once`确保了代码仅被执行了一次，同时其他调用都会阻塞，直到它完成。

* init方法外事有err检查的，所以我们不需要担心没有日志到的问题。
* 如果handler没有被调用，那么昂贵的工作就永远不会被执行 - 根据你代码事怎么部署的，可能会有非常大的好处

### The server is testable

我们的server类型非常好测试

```go
func TestHandleAbout(t *testing.T) {
    is := is.New(t)
    srv := server{
        db:    mockDatabase,
        email: mockEmailSender,
    }
    srv.routes()
    req, err := http.NewRequest("GET", "/about", nil)
    is.NoErr(err)
    w := httptest.NewRecorder()
    srv.ServeHTTP(w, r)
    is.Equal(w.StatusCode, http.StatusOK)
}
```

* 在每一个test中都创建一个server实例，如果有昂贵的东西就懒加载
* 通过调用server上的ServeHTTP，我们可以测试整个过程，报货routing和中间件，你当然也可以直接调用handler
* 使用`httptest.NewRecorder`来记录handlers做了什么
* 以上的代码用了我（作者）写的一个测试框架（用于代替Testify）

### Conclusion

我希望本文中的内容能帮助你的工作，如果你觉得哪些不对，[tweet 我们](https://twitter.com/matryer)