# Handle http request errors in Go

[原文链接](http://pliutau.com/handle-http-request-errors-in-go/)

这片短博文中，我想讨论如何在go中处理http请求的报错。我发现大家在写代码的时候，偏向于在做http请求的时候处理报错，但是这样则会丢失真正的错误。

以下爱是一个简单的http server和一个对它的Get请求

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/500", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(500)
		w.Write([]byte("NOT-OK"))
	})
	go http.ListenAndServe(":8080", nil)

	_, err := http.Get("http://localhost:8080/500")
	if err != nil {
		log.Fatal(err)
	}
}
```

这是非常简单的代码：我们运行server，然后我们对server进行请求。server会返回500返回码，我们检查返回的错误，但...

```go
if err != nil {
	log.Fatal(err)
}
```

如果我们运行以上代码，是无法捕捉到错误的。就像官方文档中所说：

> An error is returned if the Client’s CheckRedirect function fails or if there was an HTTP protocol error. A non-2xx response **doesn’t** cause an error. 

> 当客户端重定向失败或者有一个关系http协议的错误时，会返回一个error。但是一个非2xx的返回并不会造成一个 error

所以，我们应该将返回码和错误一起检查，如下:

```go
resp, err := http.Get("http://localhost:8080/500")
if err != nil {
	log.Fatal(err)
}
if resp.StatusCode != 200 {
	b, _ := ioutil.ReadAll(resp.Body)
	log.Fatal(string(b))
}
```

现在，如果我们运行代码，会打印出由于非200返回码的响应内容。这是一个很常见的新手错误。但是现在你知道，继续努力吧！