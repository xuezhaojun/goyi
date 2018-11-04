# Building High Performance APIs In Go Using gRPC And Protocol Buffers

[原文地址](https://medium.com/@shijuvar/building-high-performance-apis-in-go-using-grpc-and-protocol-buffers-2eda5b80771b)

在Go中使用 gRPC 和 Protocol Buffers 建立高性能的APIs

APIs 是现代软件的基石。api向web和app的客户端提供后端能力，同时也用跨平台的通信。一般说去要建立一个APIs,都会选择 RESTful APIs 和 Json 作为不同应用之间交互的标准。这个方法还是不错的，客户端可以很容易理解基于RESTful的Json数据。

但如果我们现在要建立一个基于云的应用，我们的微服务规模很大，并且对性能要求很高。我们的微服务之间，同样需要一个高性能通信机制。

此时，Json真的还是最好的选择吗？RESTful真的可以建立复杂的APIs吗？我们是否可以快速建立一个RESTful框架的双向流式API呢？HTTP2.0 提供了很多上一个版本没有的功能，我们在建立下一代的API时，是否要用呢？

是时候开始使用 gRPC 和 protocol buffers了。

### Introduction to Protocol Buffers

Protocol Buffers ，也简写为 protocols，是 Google 出品的，跨语言，跨平台，可扩展的机制，用于将结构化的数据序列化。

对比XML和Json, Protocol Bufers 更小，更快，更简单，同时提供了更高的性能，通过使用 Protocol Buffers, 你可以定义你的结构体，然后根据你选择的编程语言，通过 protoc 工具，来生产用于读，写结构体数据的源码。

最新的 protocol buffers 的版本为 proto3, proto3 的版本支持 C++,GO，Java, Python, Ruby 和 C#.

要生产代码，你首先得：

* [下载](https://github.com/protocolbuffers/protobuf)和安装 protoc 编译器，并将protoc的二进制文件，添加到环境变量。
* 添加你的语言需要的 protoc 插件,golang中，如下

```golang
go get -u github.com/golang/protobuf/proto
go get -u github.com/golang/protobuf/protoc-gen-go
```

### Introduction to gRPC

gRPC是高性能，开源的， remote proceduce call (RPC) 远程程度调用框架，可以运行在任何地方。它允许客户端和服务端的应用直接通信，同时让建立通信系统更加的简单。gRPC框架由Google开源。很长一段时间内，Google在它的云产品中，都用了gRPC的底层技术和概念。gRPC的设计动机和设计准则和如下：<http://www.grpc.io/blog/principles>

gRPC基于http2, 你可以建立同步和异步的通信模型。它支持传统的请求响应模型和双向流式。它构建的全双工流的功能，可以让客户端和服务端异步传输流式数据。相遇RESTful，gRPC由很多的优势。

默认情况下，gRPC使用Protocol Buffers 作为接口定义语言，也作为其底层信息的交互格式。和JSON，XML不同，Protocol Buffers 不仅仅作为信息交互的格式，同时也可以描述服务的接口以及有效负载信息的结构。在gRPC中，你可以定义服务和方法。如同传统的RPC系统中的通信一样，一个gRPC的客户端可以直接调用远程服务端的方法，如同一个本地应用。

下图为gRPC客户端和服务端交互的模式：

 ![官网图片](https://cdn-images-1.medium.com/max/800/1*X7I-IyhPdnNCsYJlm1U0Hw.jpeg)

gRPC服务端实现了服务端接口，同时运行RPC服务来处理客户端的调用。在客户端，存在一个存根，提供和server相同的方法。

### 一个Go语言实现的 gRPC/Protocol Buffers例子

我们最终的文件结构体如下：

![](https://cdn-images-1.medium.com/max/800/1*abvq5iDmLGWj4BdxRrcsyw.jpeg)

让我们试着搞一搞吧，首先你得安装gRPC：

```go
go get google.golang.org/grpc
```

#### 定义Protocol Buffers 中的 Message Types 和 Services 

让我们定义一个服务接口和一个payload message的结构体，我们要定义在一个 Protocol Buffers的文件中，文件后缀 .proto。 以下是一个样例文件：

```protobuf
syntax = "proto3";
package customer;


// The Customer service definition.
service Customer {   
  // Get all Customers with filter - A server-to-client streaming RPC.
  rpc GetCustomers(CustomerFilter) returns (stream CustomerRequest) {}
  // Create a new Customer - A simple RPC 
  rpc CreateCustomer (CustomerRequest) returns (CustomerResponse) {}
}

// Request message for creating a new customer
message CustomerRequest {
  int32 id = 1;  // Unique ID number for a Customer.
  string name = 2;
  string email = 3;
  string phone= 4;
  
  message Address {
    string street = 1;
    string city = 2;
    string state = 3;
    string zip = 4;
    bool isShippingAddress = 5; 
  }

  repeated Address addresses = 5;
}

message CustomerResponse {
  int32 id = 1;
  bool success = 2;
}
message CustomerFilter {    
  string keyword = 1;
}
```

.proto 文件开始时proto的版本，然后是包定义，我们用最新的proto3的版本作为Protocol Buffers 语言。包名为“customer”，当你通过 proto 文件生成go代码的时候，回添加这个包名。

在.proto文件内部，你定义message类型，service接口。用于定义 message 中原色的类型，由标准数据类型如 int32, float, double 和 string 。用户同样可以自定义类型。

在每一个类型的message的元素上标记的 “=1”，“=2” ，代表了每一个字段唯一的"tag"号，用于二进制编码中。未声明的字段都有一个初始零值。Protocol Buffers 的使用可见以链接:<https://developers.google.com/protocol-buffers/docs/proto3>. 

”Customer“ 定义了一个，有两个 RPC 方法的 服务:

```go
// The Customer service definition.
 service Customer { 
 // Get all Customers with a filter — A server-to-client streaming RPC.
 rpc GetCustomers(CustomerFilter) returns (stream CustomerRequest) {}
 // Create a new Customer — A simple RPC 
 rpc CreateCustomer (CustomerRequest) returns (CustomerResponse) {}
 }
```

以下是你可以定义的不同的RPC方法：

* 一个简单的, 传统请求/响应式的RPC方法，客户端通过 stub 向 RPC服务端发送一条请求，然后等待响应
* 服务端的流式RPC, 客户端发送请求给服务端，获取到一个流，然后从中读取一系列的信息。响应类型的stream 关键字，决定了一个方法是否是流式的。
* 客户端的流式RPC, 客户端通过流，将一系列的信息发送到服务端。
* 双向流式接口

上例中，Customer service 提供了两种RPC方法： 简单的 CreateCustomer 方法，和一个服务端流式的GetCustomers 方法。CreateCustomer 创建了一个新的 curstomer并以 请求/响应的方式执行。而GetCustomers通过将customers的信息通过stream的方式返回。

#### 生成客户端和服务端的Go代码

一旦你定义好了 proto 文件，下一步就是生成gRPC客户端和服务端的源码。protocol buffer compiler 和  gRPC Go 插件，可以生成客户端和服务端的代码。在application的根目录下，运行如下命令:

```go
protoc -I customer/ customer/customer.proto --go_out=plugins=grpc:customer
```

一个名为 customer.pb.go 的文件就出现了。

#### 创建 gRPC 服务端 和 客户端

下面的 main.go 文件，显示了如何使用RPC方法：

```go
ackage main

import (
	"log"
	"net"
	"strings"

	"golang.org/x/net/context"
	"google.golang.org/grpc"

	pb "github.com/shijuvar/go-recipes/grpc/customer"
)

const (
	port = ":50051"
)

// server is used to implement customer.CustomerServer.
type server struct {
	savedCustomers []*pb.CustomerRequest
}

// CreateCustomer creates a new Customer
func (s *server) CreateCustomer(ctx context.Context, in *pb.CustomerRequest) (*pb.CustomerResponse, error) {
	s.savedCustomers = append(s.savedCustomers, in)
	return &pb.CustomerResponse{Id: in.Id, Success: true}, nil
}

// GetCustomers returns all customers by given filter
func (s *server) GetCustomers(filter *pb.CustomerFilter, stream pb.Customer_GetCustomersServer) error {
	for _, customer := range s.savedCustomers {
		if filter.Keyword != "" {
			if !strings.Contains(customer.Name, filter.Keyword) {
				continue
			}
		}
		if err := stream.Send(customer); err != nil {
			return err
		}
	}
	return nil
}

func main() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	// Creates a new gRPC server
	s := grpc.NewServer()
	pb.RegisterCustomerServer(s, &server{})
	s.Serve(lis)
}
```

结构体server实现了 customer.pb.go 中的 CustomerServer 的接口。

同理，客户端代码如下：

```go

package main

import (
	"io"
	"log"

	"golang.org/x/net/context"
	"google.golang.org/grpc"

	pb "github.com/shijuvar/go-recipes/grpc/customer"
)

const (
	address = "localhost:50051"
)

// createCustomer calls the RPC method CreateCustomer of CustomerServer
func createCustomer(client pb.CustomerClient, customer *pb.CustomerRequest) {
	resp, err := client.CreateCustomer(context.Background(), customer)
	if err != nil {
		log.Fatalf("Could not create Customer: %v", err)
	}
	if resp.Success {
		log.Printf("A new Customer has been added with id: %d", resp.Id)
	}
}

// getCustomers calls the RPC method GetCustomers of CustomerServer
func getCustomers(client pb.CustomerClient, filter *pb.CustomerFilter) {
	// calling the streaming API
	stream, err := client.GetCustomers(context.Background(), filter)
	if err != nil {
		log.Fatalf("Error on get customers: %v", err)
	}
	for {
		customer, err := stream.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatalf("%v.GetCustomers(_) = _, %v", client, err)
		}
		log.Printf("Customer: %v", customer)
	}
}
func main() {
	// Set up a connection to the gRPC server.
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	// Creates a new CustomerClient
	client := pb.NewCustomerClient(conn)

	customer := &pb.CustomerRequest{
		Id:    101,
		Name:  "Shiju Varghese",
		Email: "shiju@xyz.com",
		Phone: "732-757-2923",
		Addresses: []*pb.CustomerRequest_Address{
			&pb.CustomerRequest_Address{
				Street:            "1 Mission Street",
				City:              "San Francisco",
				State:             "CA",
				Zip:               "94105",
				IsShippingAddress: false,
			},
			&pb.CustomerRequest_Address{
				Street:            "Greenfield",
				City:              "Kochi",
				State:             "KL",
				Zip:               "68356",
				IsShippingAddress: true,
			},
		},
	}

	// Create a new customer
	createCustomer(client, customer)

	customer = &pb.CustomerRequest{
		Id:    102,
		Name:  "Irene Rose",
		Email: "irene@xyz.com",
		Phone: "732-757-2924",
		Addresses: []*pb.CustomerRequest_Address{
			&pb.CustomerRequest_Address{
				Street:            "1 Mission Street",
				City:              "San Francisco",
				State:             "CA",
				Zip:               "94105",
				IsShippingAddress: true,
			},
		},
	}

	// Create a new customer
	createCustomer(client, customer)
	// Filter with an empty Keyword
	filter := &pb.CustomerFilter{Keyword: ""}
	getCustomers(client, filter)
}
```

