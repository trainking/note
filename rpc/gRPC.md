# gRPC

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
