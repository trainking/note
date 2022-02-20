# gRPC

- [gRPC](#grpc)
	- [安装依赖](#安装依赖)
	- [快速开始](#快速开始)
	- [流式RPC](#流式rpc)
		- [Proto](#proto)
		- [实践](#实践)
		- [小结](#小结)

`gRPC`是谷歌开源的跨语言，基于`http2.0`的rpc框架。`gRPC`使用`proto buffers`作为传输协议，和rpc定义工具。这份文档，只记录`Golang`的使用情况。

## 安装依赖

必须安装一下工具:

* go
* `protoc`， `proto buffers`的编译工具[proto3](https://developers.google.com/protocol-buffers/docs/proto3)
* Go的插件
```
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.26
$ go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.1
```

> 注意：要将**protoc**和**GOPATH**的bin目录（插件在其中），加入到PATH中

## 快速开始

1. 定义一分标准的`proto buffers`

```proto
syntax = "proto3";   // 声明协议版本

option go_package = "/orderrpc";  // 生成*.pb.go所在包名

package helloworld;  // rpc所在包名，会体现在rpc调用路由中

// 定义一个服务
service Greeter {      
  // 定义一个rpc
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 引用message
message HelloRequest {
  string name = 1;
}

// 引用message
message HelloReply {
  string message = 1;
}
```

2. 生成代码

```shell
 protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative .\order.proto
```

* `--go_out`指定生成message的*.pb.go文件
* `--go-grpc_out`指定生成rpc的*.pb.go文件
* `--go_opt=paths=source_relative` 指定`--go_out`使用相对路径
* `--go-grpc_opt=paths=source_relative` 指定`--go-grpc_out`使用相对路径

3. 创建服务端


```golang
var (
	port = flag.Int("port", 50051, "The server port")
)

type server struct {
	orderrpc.UnimplementedGreeterServer
}

func (s *server) SayHello(ctx context.Context, in *orderrpc.HelloRequest) (*orderrpc.HelloReply, error) {
	log.Printf("Received: %v", in.GetName())
	return &orderrpc.HelloReply{Message: "Hello " + in.GetName()}, nil
}

func main() {
	flag.Parse()

	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", port))
	if err != nil {
		log.Fatalf("failed to listen: %s", err.Error())
	}

	s := grpc.NewServer()
	orderrpc.RegisterGreeterServer(s, &server{})
	log.Printf("server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```
4. 创建客户端

```golang
type OrderStub struct {
	orderrpc.GreeterClient
}

func New(addr string) *OrderStub {
	stub := &OrderStub{}

	conn, err := grpc.Dial(addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		panic(err)
	}
	// defer conn.Close()
	stub.GreeterClient = orderrpc.NewGreeterClient(conn)
	return stub
}
```

## 流式RPC

`gRPC`是基于`HTTP/2`实现，`HTTP/2`中有两个概念，**流（stream）**和**帧（frame）**，其中帧作为`HTTP/2`通信的最小单位。`gRPC`基于`HTTP/2`协议传输，必然兼容流式传输; `gRPC`中实现了以下三种流：

1. 服务端响应
2. 客户端式请求
3. 两端双向流

### Proto

只需要在请求体中或者响应体中使用关键字`stream`，既可以定义该消息为流传输，如下:

```protobuff
syntax = "proto3";

package proto;

option go_package = "base;base";


service BaseService {
    // 服务端流式响应
    rpc ServerStream (StreamRequest) returns (stream StreamResponse){}
    // 客户端流式请求
    rpc ClientStream (stream StreamRequest) returns (StreamResponse){}
    // 双向流式
    rpc Streaming (stream StreamRequest) returns (stream StreamResponse){}
}

message StreamRequest{
  string input = 1;
}

message StreamResponse{
  string output = 1;
}
```

### 实践

基本原理是，流的发送段调用`Send`，循环发送，直到调用`SendAndClose`；接收端调用`Recv`，直到判断`io.EOF`:

```go
// 服务端
func (s *service) ClientStream(stream pb.BaseService_ClientStreamServer) error {
    output := ""
    for {
        r, err := stream.Recv()
        if err == io.EOF {
          	return stream.SendAndClose(&pb.StreamResponse{Output: output})
        }
        if err != nil {
          	fmt.Println(err)
        }
        output += r.Input
    }
}

// 客户端
func clientStream(client pb.BaseServiceClient, input string) error {
    stream, _ := client.ClientStream(context.Background())
    for _, s := range input {
        fmt.Println("Client Stream Send:", string(s))
        err := stream.Send(&pb.StreamRequest{Input: string(s)})
        if err != nil {
       	    return err
        }
    }
    res, err := stream.CloseAndRecv()
    if err != nil {
     	  fmt.Println(err)
    }
    fmt.Println("Client Stream Recv:", res.Output)
    return nil
}
```

双向流，便承担了即发也收的角色:

```go
// 客户端
func streaming(client pb.BaseServiceClient) error {
    stream, _ := client.Streaming(context.Background())
    for n := 0; n < 10; n++ {
        fmt.Println("Streaming Send:", n)
        err := stream.Send(&pb.StreamRequest{Input: strconv.Itoa(n)})
        if err != nil {
            return err
        }
        res, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
        fmt.Println("Streaming Recv:", res.Output)
    }
    stream.CloseSend()
    return nil
}


// 服务端
func (s *service) Streaming(stream pb.BaseService_StreamingServer) error {
    for n := 0; ; {
      res, err := stream.Recv()
      if err == io.EOF {
          return nil
      }
      if err != nil {
          return err
      }
      v, _ := strconv.Atoi(res.Input)
      n += v
      stream.Send(&pb.StreamResponse{Output: strconv.Itoa(n)})
	  }
}
```

### 小结

* 三种流的使用场景不同
* 双向流类似聊天室，保持了长连接