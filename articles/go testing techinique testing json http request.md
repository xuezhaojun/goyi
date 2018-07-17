# Go Testing Technique：Testing Json HTTP Request

[原文链接](https://medium.com/@xoen/go-testing-technique-testing-json-http-requests-76d9ce0e11f) 请携梯上路

你有一些http请求和json数据的代码，**那么你要测什么？怎么测？**我确信你肯定遇到过类似的问题

我会尽量让以下的例子在不失实用性的情况下，尽可能的简单。

千万不要纠结于一些小细节，比如特殊的json格式，URL 等等... 关键的部分是你发起请求的代码，和你的测试方法。

让我们开始看列子吧。

#### Example

你需要发布一条特定格式的 [NSQ](https://nsq.io) 的message:

```go
Publish(nsqdURL,msg string) error
```

这个方法将会：

1. 将 msg 字符串包装为 JSON 字符串
2. 向特定的URL发送一个POST请求
3. 如果response无法接收到，则返回一个error

#### 要测试什么？

1. 你需要测试，发布了的数据格式是否满足预期？一下实样例中的格式：

   ```json
   {
     "meta": {
       "lifeMeaning": 42
     },
     "data": {
       "message": "%{MSG}"
     }
   }
   ```

   %{MSG} 是一条字符串

2. 你需要测试，Publish() 方法做出了正确的http请求，在这个样例中，为 *POST %{nsqdUrl}/pub?topic=meaningful-topic.* 

3. 你需要测试，请求是否成功，并且返回正确的http状态，在这个例子中为 200 ok.

#### Go 测试基础

Go中提供了一个 [testing 包](https://golang.org/pkg/testing/) 用来写一些基础的测试。

我们假设 Publish() 方法是在 包 nsqpublisher中的 nsqpublisher.go 文件里。为了方便，你的测试应该在 nsqpublisher_test.go 这个文件中：

```go
package nsqpublisher
import (
 “testing”
)
func TestPublishUnreachable(t *testing.T) {
  nsqdUrl := “http://localhost:-41"
  err := Publish(nsqdUrl, “hello”)
  if err == nil {
    t.Errorf(“Publish() didn’t return an error”)
  }
}
```

 这里有几个有意思的点需要注意一下：

* 在同一个包中写测试文件，这样你就有包内的最大权限
* test方法以Test开头
* test方法接收了一个 *testing.T ，你可以通过调用它的 Errorf() 方法，让测试失败

你可以通过运行 `go test` 来跑测试。

测试无法通过，因为目前还没有具体的Publish()方法，让我们写一个吧。

#### 第一个（有问题的）实现

我们先以一个有问题的实现版本开始：

```go
package nsqpublisher
import (
  "net/http"
)
func Publish(nsqdUrl, msg string) error {
  _, err := http.Get(nsqdUrl)
  return err
}
```

测试不会通过，并且会返回error。

#### 一个 HTTP test 服务器

第一个测试虽然可用，但是你要怎么在没有运行NSQ server（或者其他任何服务器）的情况下，来做 HTTP的请求呢？

这个问题的答案就是使用 [httptest 包](https://golang.org/pkg/net/http/httptest/) ,这个包使得创建一个 测试用的 http server 特别的容易。

让我们看看，如果给 当nsq server 没有返回 200 ok 的情况写测试用例。

```go
import (
  "testing"
  "net/http"
  "net/http/httptest"
)
func TestPublishWrongResponseStatus(t *testing.T) {
  ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusServiceUnavailable)
  }))
  defer ts.Close()
  nsqdUrl := ts.URL
  err := Publish(nsqdUrl, "hello")
  if err == nil {
    t.Errorf("Publish() didn’t return an error")
  }
}
```

好了，稍微有一点复杂：

* 我们通过 http.NewServer 创建了一个 测试用的server
* 我们通过 ts.URL 获取到了这个server 的URL
* 这个server通过 w.WriteHeader() 方法来返回一个 503 问题
* 这个server有对 r 就是 *http.Request 的权限（本例中没有用到）
* 这个server会在测试结束的时候，关闭掉，我们使用的是 `defer ts.Close()` 方法

如果你运行这个测试，测试会失败： Publish() 发起了一个请求，但是没有检查响应的状态码。修复起来很简单：

```go
package nsqpublisher
import (
  "net/http"
  "fmt"
)
func Publish(nsqdUrl, msg string) error {
  resp, err := http.Get(nsqdUrl)
  if err != nil {
    return err
  }
  if resp.StatusCode != http.StatusOK {
    return fmt.Errorf(“nsqd didn’t respond 200 OK: %s”, resp.Status)
  }
  return nil
}
```

#### 测试请求

现在我们可以轻松来测试 Publish() 是否发起了正确的请求了。

```go
func TestPublishOK(t *testing.T) {
  ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
if r.Method != "POST" {
      t.Errorf(“Expected ‘POST’ request, got ‘%s’”, r.Method)
    }
    if r.URL.EscapedPath() != "/pub" {
      t.Errorf("Expected request to ‘/pub’, got ‘%s’", r.URL.EscapedPath())
    }
    r.ParseForm()
    topic := r.Form.Get("topic")
    if topic != "meaningful-topic" {
      t.Errorf("Expected request to have ‘topic=meaningful-topic’, got: ‘%s’", topic)
    }
  }))
  defer ts.Close()
  nsqdUrl := ts.URL
  err := Publish(nsqdUrl, "hello")
  if err != nil {
    t.Errorf("Publish() returned an error: %s", err)
  }
}
```

运行测试，你可以看到以下内容：

```shell
$ go test
 — — FAIL: TestPublishOK (0.00s)
         nsqpublisher_test.go:35: Expected ‘POST’ request, got ‘GET’
         nsqpublisher_test.go:38: Expected request to ‘/pub’, got ‘/’
         nsqpublisher_test.go:44: Expected request to have ‘topic=meaningful-topic’, got: ‘’
FAIL
exit status 1
```

实用吧，我们现在有了一个可以精确的告诉我们 Publish() 哪里有问题。没错，HTTP 的请求对应的 path 是不对的，通过携带的 verb 也是有问题的。

其中是怎么做到的呢?

* 测试用server的 hanlder 有对 t 的曲线，可以让测试不通过
* 测试用server会检测 http request
* 通过 r.Method 来获取请求方法
* 通过 r.URL.EscapedPath() 获取到 request path
* 通过 r.ParseForm() 或者 r.Form.Get("topic")

以下修改为了让测试通过：

```go
package nsqpublisher
import (
  "net/http"
  "fmt"
)
const (
  TOPIC = "meaningful-topic"
)
func Publish(nsqdUrl, msg string) error {
  url := fmt.Sprintf("%s/pub?topic=%s", nsqdUrl, TOPIC)
  resp, err := http.Post(url, "application/json", nil)
  if err != nil {
    return err
  }
  if resp.StatusCode != http.StatusOK {
    return fmt.Errorf("nsqd didn’t respond 200 OK: %s", resp.Status)
  }
  return nil
}
```

为了让测试用例通过，我们可以修改right url

```go
package nsqpublisher
import (
  "net/http"
  "fmt"
)
const (
  TOPIC = "meaningful-topic"
)
func Publish(nsqdUrl, msg string) error {
  url := fmt.Sprintf("%s/pub?topic=%s", nsqdUrl, TOPIC)
  resp, err := http.Post(url, "application/json", nil)
  if err != nil {
    return err
  }
  if resp.StatusCode != http.StatusOK {
    return fmt.Errorf("nsqd didn’t respond 200 OK: %s", resp.Status)
  }
  return nil
}
```

测试这下可以通过了。

#### 测试我们要发送的数据

我们的Publish() 方法目前还有问题，它没有推送任何数据。request body 应该是特定格式的JSON。

我们要怎么测试它？你可能已经猜到，我们需要在test server中读取request body，解析然后测试它是否有正确的属性。

我会使用go-simplejson包。

```go
func TestPublishOK(t *testing.T) {
  msg := "Test message"
  ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    if r.Method != "POST" {
      t.Errorf("Expected ‘POST’ request, got ‘%s’", r.Method)
    }
    if r.URL.EscapedPath() != "/pub" {
      t.Errorf("Expected request to ‘/pub’, got ‘%s’", r.URL.EscapedPath())
    }
    r.ParseForm()
    topic := r.Form.Get("topic")
    if topic != "meaningful-topic" {
      t.Errorf("Expected request to have ‘topic=meaningful-topic’, got: ‘%s’", topic)
    }
    reqJson, err := simplejson.NewFromReader(r.Body)
    if err != nil {
      t.Errorf("Error while reading request JSON: %s", err)
    }
    lifeMeaning := reqJson.GetPath("meta", "lifeMeaning").MustInt()
    if lifeMeaning != 42 {
      t.Errorf("Expected request JSON to have meta/lifeMeaning = 42, got %d", lifeMeaning)
    }
    msgActual := reqJson.GetPath("data", "message").MustString()
    if msgActual != msg {
      t.Errorf("Expected request JSON to have data/message = ‘%s’, got ‘%s’", msg, msgActual)
    }
  }))
  defer ts.Close()
  nsqdUrl := ts.URL
  err := Publish(nsqdUrl, msg)
  if err != nil {
    t.Errorf("Publish() returned an error: %s", err)
  }
}
```

最关键的东西是：

* 通过读取request body reader 来创建了一个 simplejson 的对象 reqJson
* 我们通过 simpleJson的GetPath来判断是否有对应的Json熟悉
* 我们通过int/string by using *MustInt()/MustString()* 来转化值

运行测试，会失败：

```go
$ go test
 — — FAIL: TestPublishOK (0.00s)
         nsqpublisher_test.go:52: Error while reading request JSON: EOF
         nsqpublisher_test.go:56: Expected request JSON to have meta/lifeMeaning = 42, got 0
         nsqpublisher_test.go:60: Expected request JSON to have data/message = ‘Test message’, got ‘’
```

Publish() 还没发布任何数据，让我们解决这个问题

#### 发布正确的json数据

```go
package nsqpublisher
import (
  "fmt"
  "net/http"
  "strings"
)
const (
  LIFE_MEANING = 42
  TOPIC        = "meaningful-topic"
)
func Publish(nsqdUrl, msg string) error {
  url := fmt.Sprintf("%s/pub?topic=%s", nsqdUrl, TOPIC)
  payload := fmt.Sprintf(`
    {
      "meta": {
        "lifeMeaning": %d
      },
      "data": {
        " “message”: "%s"
      }
    }`, LIFE_MEANING, msg)
  resp, err := http.Post(url, "application/json", strings.NewReader(payload))
  if err != nil {
    return err
  }
  if resp.StatusCode != http.StatusOK {
    return fmt.Errorf("nsqd didn’t respond 200 OK: %s", resp.Status)
  }
  return nil
}
```

NOTE： 这个简单的例子中，我仅仅是用了 fmt.Sprintf() 这个方法来创建Json，更复杂的情况下，我建议使用 simplejson

到了这一步，Publish() 终于可以使用正确的数据做一次POST请求到正确的URL，测试全部通过。

#### 总结

如你所见，要测试发送HTTP请求在go中非常容易。

多亏 httptest 让测试中可以建立对应的简单 http server。

我希望这篇文章有用，并能帮助你写出更好的代码。如果你有更好的方法，请告知我。