---
title: gRPC之interceptor
date: 2020-11-08 21:13:25
tags:
    - gRPC
categories:
    - gRPC
---

在Web服务中，除了实际的业务代码，经常还需要实现统一记录请求日志，权限管理或异常处理等功能，这些在web框架Gin或Django中可通过middleware实现，而gRPC中则可使用interceptor，对rpc请求或响应进行拦截处理。

gRPC服务端跟客户端均可实现各自的拦截器，根据rpc的两种请求方式可分为两种。
- Unary Interceptor（一元拦截器）
- Stream Interceptor（流式拦截器）

## 一元拦截器

对于一元服务器拦截器，只需要定义`UnaryServerInterceptor`方法即可，其中，`handler(ctx, req)`即调用rpc方法。

```go
type UnaryServerInterceptor func(
    ctx context.Context,    // rpc上下文
    req interface{},        // rpc请求参数
    info *UnaryServerInfo,  // rpc方法信息
    handler UnaryHandler    // rpc方法本身
) (interface{}, error){
	return handler(ctx, req)
}
```

而对于一元客户端拦截器，一样需要定义一个方法`UnaryClientInterceptor`，其中执行`invoker()`才真正请求rpc。

```go
type UnaryClientInterceptor func(
    ctx context.Context,  // rpc上下文
    method string,        // 调用方法名
    req,                  // rpc请求参数
    reply interface{},    // rpc响应结果
    cc *ClientConn,       // 连接句柄
    invoker UnaryInvoker, // 调用rpc方法本身
    opts ...CallOption    // 调用配置
) error {
    return invoker(ctx, method, req, reply, cc, opts...)
}
```

一元拦截器的实现，根据调用handler或invoker的前后，可分为三部分：调用前预处理，调用rpc方法，调用后处理。

## 流式拦截器

流式拦截器的实现，与一元拦截器一致，实现提供的方法即可，方法参数含义如下：

```go
type StreamServerInterceptor func(
    srv interface{},        // rpc请求参数
    ss ServerStream,        // 服务端stream对象
    info *StreamServerInfo, // rpc方法信息
    handler StreamHandler   // rpc方法本身
) (err error){
    return handler(src, ss)
}
```

```go
type StreamClientInterceptor func(
    ctx context.Context,   // rpc上下文
    desc *StreamDesc,      // 流信息
    cc *ClientConn,        // 连接句柄
    method string,         // 调用方法名
    streamer Streamer,     // 调用rpc方法本身
    opts ...CallOption     // 调用配置
)(ClientStream, error){
    // 流操作预处理
    clientStream, err := streamer(ctx, desc, cc, method, opts...)
    // 根据某些条件，通过clientStream拦截流操作
    return clientStream, err
}
```

与其他拦截器不同，客户端流式拦截器的实现分为两部分，流操作预处理和流操作拦截，其不能在事后进行rpc方法调用和后处理，只能通过ClientStream对象进行流操作拦截，例如根据特定的metadata，调用`ClientStream.CloseSend()`终止流操作。

## 举个例子

这里我们将上述四种拦截器都写一个简单输出请求日志的demo，看看实际效果

demo目录结构如下

```
rpc
├── base.proto
├── base
│   ├── base.pb.go
│   └── base_grpc.pb.go
├── client
│   └── main.go
├── server
│    └── main.go
├── go.mod
├── go.sum
```

base.proto文件如下

```protobuf
syntax = "proto3";

package proto;

option go_package = "base;base";


service BaseService {
  rpc GetTime (TimeRequest) returns (TimeResponse){}
  rpc Streaming (stream StreamRequest) returns (stream StreamResponse){}
}

message TimeRequest {}

message TimeResponse {
  string time = 1;
}

message StreamRequest{
  string input = 1;
}

message StreamResponse{
  string output = 1;
}
```

执行命令`protoc --go_out=. --go-grpc_out=. base.prto`生成对应的pb文件。

server.go

```go
package main

import (
	"context"
	"log"
	"io"
	"net"
	"strconv"
	"time"

	pb "rpc/base"

	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

type service struct {
	pb.UnimplementedBaseServiceServer
}

func main() {
	listen, err := net.Listen("tcp", ":50051")
	if err != nil {
		fmt.Println(err)
	}
	s := grpc.NewServer(grpc.UnaryInterceptor(UnaryServerInterceptor), grpc.StreamInterceptor(StreamServerInterceptor))
	reflection.Register(s)
	pb.RegisterBaseServiceServer(s, &service{})
	s.Serve(listen)
}

func UnaryServerInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
	log.Println("start unary")
	resp, err = handler(ctx, req)
	log.Printf("end unary %v\n", resp)
	return resp, err
}

func StreamServerInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
	log.Println("before stream")
	err := handler(srv, ss)
	log.Println("after stream")
	return err
}

// 具体接口实现
func (s *service) GetTime(ctx context.Context, in *pb.TimeRequest) (*pb.TimeResponse, error) {
	now := time.Now().Format("2006-01-02 15:04:05")
	return &pb.TimeResponse{Time: now}, nil
}

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
        log.Println(v)
		n += v
		stream.Send(&pb.StreamResponse{Output: strconv.Itoa(n)})
	}
}
```

client.go

```go
package main

import (
	"context"
	"io"
	"log"
	"strconv"

	pb "rpc/base"

	"google.golang.org/grpc"
)

func main() {
	conn, err := grpc.Dial(":50051", grpc.WithInsecure(), grpc.WithBlock(), grpc.WithUnaryInterceptor(UnaryClientInterceptor), grpc.WithStreamInterceptor(StreamClientInterceptor))
	if err != nil {
		log.Println(err)
	}
	defer conn.Close()

	c := pb.NewBaseServiceClient(conn)
	
    // 依次执行unary，stream
    _, err = c.GetTime(context.Background(), &pb.TimeRequest{})
	if err != nil {
		log.Fatal(err)
	}
    print("===============\n")
    streaming(c)
}

func UnaryClientInterceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
	log.Println("before unary")
	err := invoker(ctx, method, req, reply, cc, opts...)
	log.Printf("end unary %v\n", reply)
	return err
}

func StreamClientInterceptor(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error) {
	log.Println("before stream")
	clientStream, err := streamer(ctx, desc, cc, method, opts...)
	log.Println("check metadata")
	return clientStream, err
}

func streaming(client pb.BaseServiceClient) error {
	stream, _ := client.Streaming(context.Background())
	for n := 0; n < 10; n++ {
		log.Println("Streaming Send:", n)
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
		log.Println("Streaming Recv:", res.Output)
	}
	stream.CloseSend()
	return nil
}
```

rpc目录下依次执行`go run server/main.go` 和 `go run client/main.go`，输出效果如下

![输出结果](/images/grpc-interceptor.png)

可以明显看出StreamClientInterceptor则是在流处理开始时就输出了两次日志，其余三种拦截器则在请求前后输出两次。

## 小结

如果需要使用多个拦截器，grpc-go中提供了相应的四种链接器

- `grpc.ChainUnaryInterceptor(i ...UnaryServerInterceptor)`
- `grpc.ChainStreamInterceptor(i ...StreamServerInterceptor)`
- `grpc.WithChainUnaryInterceptor(i ...UnaryClientInterceptor)`
- `grpc.WithChainStreamInterceptor(i ...StreamClientInterceptor)`

如果grpc版本过老，可能还未提供chain api，可以使用第三方库[grpc-ecosystem/go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware)。除了链接器，库中还提供了许多常用的拦截器，例如grpc_zap，grpc_recovery等。当然，特殊需求也可以通过实现对应方法，实现自定义interceptor。
