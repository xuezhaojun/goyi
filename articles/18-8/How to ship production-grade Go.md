# How to ship production-grade Go

[原文链接](https://www.oreilly.com/ideas/how-to-ship-production-grade-go)

> 如果写出生产级别的代码

生产级别的Go应用和普通的Go应用不同。记住，最大的区别是，你的代码和你在生产环境的代码就是对待fail的方式，生产级别的代码应该注意到这个区别，预防或者计划它。

如何将你的代码转化为生产级别的代码呢？

本文将用5点让你的代码可以在生产环境在运行，目标是让你的代码，稳定，可调试，可上线。

### Wrap Errors

在生产环境的Go应用，会对errors和panic做统计。这个节将会讨论如何写代码让error的返回和处理更加的好用，下一节会讨论panic。

首先，能写返回error的，就别panic（Don't panic）。Errors 表明了出现异常状态 ，而panic则表示无法恢复的状态，Errors应该处理，而panic则应该退出程序。

当你处理一个`error`的时候，要为调试注释上有用的上下文信息。Dave Cheney的[errors](https://github.com/pkg/errors)  包就非常适合，当然你也可以选用其他的。

以下是使用`error`包的注释error的样例：

* 比如你有一个返回int和error的方法`getCount`。使用`errors.New`来创建一个新的error。它会用提供的信息创建一个error，并用可追溯的信息注释它。

  ```go
  // Import Dave Cheney's errors package
  "import github.com/pkg/errors"
  
  func getCount(key) (int, error) {
    if key <= 0 {
      // Create an annotated error.
      err := errors.New("invalid key")
      return 0, err    
    }
    ...
    return count, nil
  }
  ```

* 现在,`canProceed`方法调用了`getCount`，`canProceed`会检查error，如果error不是nil，则会处理或者上传它。

  ```go
  func canProceed(key int) bool {
    count, err := getCount(key)
    if err != nil {
      // Handle or propagate err.
    } else {
      return count < threshold
    }
  }
  ```

  后面我们有讨论如何选择处理error还是上传error。目前，我们先看一下本例中是如何使用`error`包实现两者的。

* 如何`canProceed`将err上传到它的caller，首先需要将error打包

  `errors.Wrap`会返回一个有此处追溯信息，和提供的信息的新error

  ```go
  "import github.com/pkg/errors"
  
  func canProceed(key int) (bool, error) {
    count, err := getCount(key)
    if err != nil {
      // Wrap err with a message and stack trace before propagating it.
      return false, errors.Wrap(err, “getCount failed”)
    }
    return count < threshold, nil
  }
  ```

* 同样，`canProceed`也可以处理掉error - 打日志出来，或者递增error的计数。

  ```go
  func canProceed(key int) bool {
    count, err := getCount(key)
    if err != nil {
      // Handle err by logging it.
      // If err was created by the errors package,
      // this would print the annotations as well.
      fmt.Printf(“%+v”, err)
      return false
    }
    return count < threshold
  }
  ```

  这个注释威力在debug的时候就很明显了。打应一个带注释的error，是否打应err的追溯信息是可选的：

  ```go
  // To print the annotated error without the stack trace:
  fmt.Printf("%v", err)
  getCount failed: invalid key
  
  // To print the stack trace as well, use %+v formatting flag
  fmt.Printf("%+v", err)
  ```

  还有一点就是 -- 只处理error一次。在上面的例子中，一点err被打印出来，就应该被考虑为已处理过。如果需要，错误可以返回给canProceed的调用方，已表明另一个返回值是否需要消耗，但是理想情况下，是否不需要再次处理error的。

### Report Panics

这一章节讨论panics。不同于errors, panics 不应该被程序处理（handled） -- 他们表示出现了一种不可恢复的状态，应该中断程序的状态。也就是说，如果将生产环境下产生的panic通过一个 [Slack](https://slack.com/)  的管道报告出来将会非常的有用。记住，panic是bug，而且是严重的bug - 但同时他们也应该很容易定位和解决。

首先，让我们讨论一下对于panic，我们要做什么，然后，怎么做。

一个未检验的panic会在程序已非0状态结束之前，打印出一条错误信息和goroutine的栈信息到`stderr`。`staerr`日志对于生产环境的panic的及时处理是非常不足的，所以需要将它们日志到向 [Sentry](https://sentry.io/welcome/)   这样服务中来通知你。同样，你也可以通过配置你的应用，让它可以通过Slack的channel来直接通知你，但是能用 Sentry还是用Sentry吧。

至于怎么报告panics：

* 使用内建的recover方法来报告你的应用产生的panic

  `recover`捕获了当前goroutine的panic值；如果goroutine崩溃了，那么panic值就不会为空。`recover`只会在`defer`方法中有效，它会快速的将崩溃的goroutine的控制恢复。一旦调用了`recover`，recover后的程序就不会执行了。所以`recover`应该写在最顶层的function的`defer`中。

  以下是一个例子：

  ```go
  // postToSlack creates a message with the captured panic, timestamp, and hostname,
  // and posts it to the configured Slack channel.
  func postToSlack(panic interface{}) {
    ...
  }
  
  func reportPanics() {
    // Capture the panic if there is one, and report it.
    if panic := recover(); panic != nil {
      postToSlack(panic)
    }
  }
  
  // Version without panic reporting; this is not what you want!: 
  func runWithoutPanicReporting() {
    // myFunc encapsulates the core logic to do something.
    // Run it in a separate goroutine.
    go myFunc()
  }
  
  // Version with panic reporting; this is what you want:
  func runMyFuncPanicReporting() {
    go func() {
      // Defer reportPanics before calling myFunc --  
      // if myFunc panics, reportPanic will capture and report the panic right before
      // the goroutine exits.
      defer reportPanics()
      myFunc()
    }()
  }
  ```

  以上的这个方案可以用于 HTTP server 应用中，任何request goroutine中的panic都可以报告出来。同时介绍一个panic reporting的中间件，使用中间件将在调用下一个handler之前就 `defer reportPanics()`，要确保中间件链路上，第一个使用这个中间件，这样才能保证所有的panic都会汇报出来：

  ```go
  func PanicReporterMiddleware(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
    defer reportPanics()
    next(w, r)
  }
  
  // server.go
  func runServer() {
    // Set up your routes.
    router := setUpRouter()
    
    // Chain the PanicReporter middleware first:
    // this example uses the negroni middleware library, but you can set this up
    // in any middleware library.
    n := negroni.New()
    n.Use(negroni.HandlerFunc(PanicReporterMiddleware))
    n.UseHandler(router)
  
    // Run the HTTP server.
    http.ListenAndServe(":3001", n)
  }
  ```

* 以上的defered `recover`方法，是没法报告出由第三方库创建的goroutine的 panic。为了能将这些Panic也报告出来，可以通过使用 HashiCorp 的  [panicwrap](https://github.com/mitchellh/panicwrap)  包，来报告应用级别的panic，或者使用Supervisor来查看。

  你可以很直接的使用`panicwrap`，但是如果把它配合正确，则需要你对它如何工作，已经你的程序如何运行要有比较深的理解。不过这些内容不再本文的讨论范围，我建议[读读代码](https://github.com/mitchellh/panicwrap/blob/master/panicwrap.go) ，然后根据你的需求使用它。

### Use structured logging

这一节有关于logging（日志），能否洞察生成环境下的程序的行为，这一节至关重要。

结构化的logging是指，使用 key-value 对的形式，而非自由文本。标准库 `log` 包值提供后者，但是你应该使用第三包来提供前者。

在我们探究为什么要用结构化的日志而不是非结构化的日志之前，先看看以下的示例：

```go
log.log.Printf("Redirecting userId %d to %s", 1, "www.google.com")
```

输出为：

```go
2017/03/25 17:00:00 Redirecting userId 1 to www.google.com
```

而使用结构化的日志包，比如 `logrus`:

```go
import log "github.com/Sirupsen/logrus"

func main() {

  log.SetFormatter(&log.JSONFormatter{})

  log.WithFields(log.Fields{

    "userId": 1,
    "server": "www.google.com",

  }).Info("Redirecting user")

}
```

其输出是：

```json
{"level":"info","msg":"Redirecting user","server":"www.google.com","time":"2017-03-25T17:00:00-08:00","userId":1}
```

没错，非结构化的日志更易读，但是这些不确定的，字符串形式的日志，很难进行筛选，查询或处理。相反，虽然结构化的日志不易读，但是结构化的格式，意味着你的日志可以被序列化然后处理；使用工具（比如 [EKL](https://www.elastic.co/)）来进行日志分析。

go中著名的结构化的第三方库是 `logrus`， 也是上面那个例子中用的，其他的还有`zap`和`log15`。这些库支持很多结构化类型（json以及其他），同时也支持很多标准库不具有的特性 - 比如日志等级等

举个例子，使用这些库，你可以遍历一个ruquest handler的日志，其中都包含有 `userId`,`requestId` 和 `endpoint` 字段：

```go
import log "github.com/Sirupsen/logrus"

func init() {
  log.SetFormatter(&log.JSONFormatter{})
}

func requestHandlerToDoSomething() {
  // Get request parameters
  if userId, err := getRequestParams(); err != nil {
    return http.Error(w, “Request must contain a valid userId”, http.StatusBadRequest) 
  }

  // Include the userId, requestId, and endpoint to all log statements that 
  // use this logger.
  logger := log.WithFields(log.Fields {
    “userId”: userId,
    "requestId": ...,
    "endpoint": ...,
  })
  
  logger.Info("Request started")

  if err := checkIfUserCanDoSomething(); err != nil {
    logger.Error("Unauthorized user")
    return http.Error(w, “Unauthorized access”, http.StatusForbidden)
  }

  if err := DoSomething(); err != nil {
    logger.Error("DoSomething failed")
    return http.Error(w, "server error", http.StatusInternalServerError)
  }

  logger.Info("Request succeeded")
  w.WriteHeader(http.StatusOK)
}

{"endpoint":"GET /doSomething","level":"info","msg":"Request started","requestId":1,"time":"2017-03-25T17:00:00-07:00","userId":1}

{"endpoint":"GET /doSomething","level":"error","msg":"DoSomething failed","requestId":1,"time":"2017-03-25T17:18:48-07:00","userId":1}
```

### Ship application metrics

Instrumentation （仪表）让你随时可以回答关于你的应用表现以及性能的相关问题，一个完善的仪表包括收集和分析系统和应用的行为和性能，并且一直监控它们。其中系统的性能本文不涉及，我们在这一节，只聊你的应用性能 - 也即，你的应用要expose(导出/暴露)出的指标数据。

首先一点是，你要收集哪些指标数据(metrics)？

这依赖于应用的类型，还有你需要回答的问题是什么。举例，如果你在运行一个HTTP service，问题的最小集为：request返回一个err response的百分比是多少？百分之99的响应时间是多少？更甚，你可以收集每一个终端的信息，并聚合使用。你还可以收集其他信息，比如在任意给定时间内的有多少用户限速了，或者访问一个服务要多节？想知道这些，建一个metric点吧！

下一个问题是，你收集到的指标数据，要发到哪里？

有很多的指标系统，比如 [Datadog](https://www.datadoghq.com/), [Prometheus](https://prometheus.io/) ,[StatsD + Graphite](https://graphiteapp.org/#integrations)  都支持存储，聚合，以及图形化时间分片的指标数据。它们都有提供Go的库，让你在你的application中收集和发送指标数据。牢记这些系统的选择主要看你的需求是什么 - 你想花多长时间，你是否想自己搭建  - 随着需求的变化，选择也跟着会变。

好好建立这些系统，在线文档都很清楚，一旦你决定好，那么开始使用也不会有什么问题。

### Test more than you think you should

你肯定，必须给你在生产环境的代码写测试，这一节会讨论如何通过写更多类型，更加严苛的测试来让测试更进一步。

在一般go项目中，test扮演一个包里的单元测试。比如`api`包中有`server.go`和`users.go`，那么对应的`_test.go` 文件则为：`server_test.go`和`users_test.go`。对于包中的每一个方法，都有一个或多个对应的`Test`方法。这个虽然看起来不错，但是其实只是很小的一部分，并且几乎都不满足测试正式生产环境的代码。

* 如果你的包提供一个public接口 - 而且这个接口可能之后的你代码中使用 - 就要像客户一样去测试这个接口。你需要在当前这个包外去写一个测试文件，来测试你的包暴露出来的方法；包内的单元测试则可以用于测试内部方法，private的状态等

* 写对接处的测试
  这一点需要建立在上一点的基础上，对接处的测试是用来测试你的组件之间的交互。
  使用上面的例子，你有一个内部的admin服务，和api服务通信，并获取用户的列表。这两个服务的代码分布于两个独立的包中，并且有独立的包内单元测试，已经包外接口测试，但是你应该额外的添加一个测试集，用于测试两个模块之间的交互 - 比如admin服务的request格式是否满足api服务的需求呢？api服务返回的response admin服务是否能处理呢？
  针对于上面的例子，我建议的包结构应如下：

  ```go
  api/
    server.go
    server_test.go
    users.go
    users_test.go
  
  admin/
    server.go
    server_test.go
    collector.go
    collector_test.go
  
  tests/
    api_test.go
    admin_test.go
    integration_test.go
  ```

  作为之前两种测试类型的补充，端对端的测试对于复杂系统来说是无价的。我们这里不会讨论，因为这个一般是对应分布式系统，不在application应用的范围之内。但是footnote中会包含他们的信息。

* 使用正确的测试工具

  Go本身有一大堆的测试工具，一定要用！
  写了一个有复杂输入，或者用户提供输入的应用？Dimitry Vyokov 的[go-fuzz](https://github.com/dvyukov/go-fuzz#go-fuzz-randomized-testing-for-go)  包，可以对Go程序做极好的模糊测试（随机测试）。

  需要检查代码的标准输出？只要在你的测试中，使用标准库中的 [Example functions](https://golang.org/pkg/testing/#hdr-Examples)  即可

  想测试HTTP的客户端和服务端?标准库的 [testing/httptest](https://golang.org/pkg/net/http/httptest/) 是你的好小伙伴。

  要测试IO读写？看看标准库中的 [testing/iotest](https://golang.org/pkg/testing/iotest/) 

  根本说不完...所以，如果你发现生成环境下未测试的代码，就立马停下。不管你想测试什么，几乎总有一个对应Go包让你使用

* 确保你的代码包含了所有重要的代码路径。运行`go test -cover` 来执行包中的测试用例，这样会自动计算出包内的代码在测试中的覆盖程度是怎样的。使用`go test -coverprofile=<filename>` 来产生覆盖程度的分析文件，然后你可以使用go的cover命令来分析。

测试要早，测试要频繁 - 没告诉你这句话的testing介绍都是不完整的。你的codeview工具应该满足 - 测试样例要在每次review都执行，必须包含一个测试覆盖报告。构建工具需要运行你的测试用例，并且在测试失败的时候fail掉。

---

我没想用很多很详细的点来，以上5点知识实现生产级别代码的最小集合，并且建立在已经有良好文档和测试习惯的情况下。

- Wrap errors
- Report panics
- Use structured logs
- Ship application metrics
- Write more tests than you think you should.

 ### Footnotes

1. Palantir 的 [stacktrace](https://github.com/palantir/stacktrace) 是另一个用栈追溯的信息包装error的包，不过前文中的  [errors](https://github.com/pkg/errors)  包有更富的api和使用范围也更广

2. 如何选择上报error还是处理error：第一要素是需考虑调用链中，error在哪里最有用的。比如：如果`canProceed`返回了一个error - 它的调用者需要根据发生了错误还是发生了count大于threshold的情况来发送不同的HTTP response。这样调用者就即需要处理错误，又需要采取一些其他的操作，比如logging；而`canProceed`则不用。

   ```go
   func MyRequestHandler(w http.ResponseWriter, r *http.Request) {
     // Derive key from the request.
     ...
     
     // Is the user is rate-limited?
     proceed, err := canProceed(key)
     
     // Could not compute count; examine err to send an appropriate response.
     if err != nil {
       fmt.Printf("Could not compute count. err: %v", err)
       
       // HTTP 400 for an invalid key error, HTTP 500 for all others.
       status := statusFromErr(err)
       http.Error(w, http.StatusText(status), status)
       return
     }
     
     // count >= threshold; send a rate-limited response.
     if !proceed {
       fmt.Println("User rate-limited", ...)
       
       status := http.StatusTooManyRequests
       http.Error(w, http.StatusText(status), status)
       return
     }
     
     // Perform request.
   }
   ```

3. Sentry  除了通知panic以外，还提供了很多功能 - 你可以使用它来分析stack traces, track trends ，确定bug影响等

4. Upstart, [Supervisor](http://supervisord.org/), [runit](http://smarden.org/runit/)  都是非常常见的进程管理器，可以利用它们来查看那些进程异常退出了。

5. 有很多的日志管理服务可以选择，比如之前提到的ELK和[Graylog](https://www.graylog.org/) 都可以自己搭建在自己的电脑。如果你不要搭建，而且愿意付钱， 可以选择广泛使用的 [hosted ELK](https://www.elastic.co/cloud), [Loggly](https://www.loggly.com/) and [Splunk](https://www.splunk.com/)  ；[Honeycomb](https://honeycomb.io/)  虽然是新出的，但是看起来也挺靠谱的。

6. metrics指标数据系统很多：Prometheus （可以自己搭建），StatsD + Graphite ，Datadog ， [Honeycomb.io](https://honeycomb.io/) 等

7. 不错的code review 工具，[Phabricator](https://www.phacility.com/),  [Gerrit](https://www.gerritcodereview.com/). 连续构建的工具 [Jenkins](https://jenkins.io/) and [CircleCI](https://circleci.com/) 

