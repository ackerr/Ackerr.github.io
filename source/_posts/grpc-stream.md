---
title: gRPC之stream 
date: 2020-09-06 23:48:06
tags:
    - gRPC
categories:
    - gRPC
---

HTTP/2中有两个概念，流（stream）与帧（frame），其中帧作为HTTP/2中通信的最小传输单位，通常一个请求或响应会被分为一个或多个帧传输，流则表示已建立连接的虚拟通道，可以传输多次请求或响应。每个帧中包含**Stream Identifier**，标志所属流。HTTP/2通过流与帧实现多路复用，对于相同域名的请求，通过**Stream Identifier**标识可在同一个流中进行，从而减少连接开销。 而gRPC基于HTTP/2协议传输，自然而然也实现了流式传输，其中gRPC中共有以下三种类型的流
1. 服务端流式响应
2. 客户端流式请求
3. 两端双向流式

本篇主要讲讲如何实现gRPC三种流式处理。

## Proto

通过在请求体或响应体前添加关键词`stream`，即可定义该消息体为流传输，`base.proto`如下所示

```protobuf
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

执行`protoc --go_out=. --go-grpc_out=. stream.prto`，生成对应的Go Stub。

## 基础代码

这里先将附上代码模板
server.go
```go
package main

import (
	"context"
	"fmt"
	"net"
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
	s := grpc.NewServer()
	reflection.Register(s)
	pb.RegisterBaseServiceServer(s, &service{})
	s.Serve(listen)
}

func (s *service) ClientStream(stream pb.BaseService_ClientStreamServer) error {
    return nil 
}

func (s *service) ServerStream(in *pb.StreamRequest, stream pb.BaseService_ServerStreamServer) error {
	  return nil
}

func (s *service) Streaming(stream pb.BaseService_StreamingServer) error {
    return nil
}

```
client.go
```go
package main

import (
	"context"
	"fmt"
	"io"
	"strconv"

	pb "rpc/base"

	"google.golang.org/grpc"
)

func main() {
	conn, err := grpc.Dial(":50051", grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		fmt.Println(err)
	}
	defer conn.Close()

	c := pb.NewBaseServiceClient(conn)
}

func clientStream(client pb.BaseServiceClient, input string) error {
	  return nil
}

func serverStream(client pb.BaseServiceClient, r *pb.StreamRequest) error {
	  return nil
}

func stream(client pb.BaseServiceClient) error {
	  return nil
}

```

## 服务端流式响应

与普通的gRPC代码不同的是，普通gRPC一次请求只会返回一次响应，而服务端流式响应通过调用`stream.Send()`，返回多次``StreamResponse`

```go
// 服务端
func (s *service) ServerStream(in *pb.StreamRequest, stream pb.BaseService_ServerStreamServer) error {
    input := in.Input
    var output string
    for i := 0; i < len(input); i++ {
      	output = fmt.Sprintf("index: %d, result: %s", i, string(input[i]))
     		stream.Send(&pb.StreamResponse{Output: output})
    }
    return nil
}
```
客户端代码如下，通过for循环调用`stream.Recv()`，等待接收服务端响应并阻塞，直到出错或流结束。

```go
// 客户端
func serverStream(client pb.BaseServiceClient, r *pb.StreamRequest) error {
    fmt.Println("Server Stream Send:", r.Input)
    stream, _ := client.ServerStream(context.Background(), r)
    for {
        res, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
        fmt.Println("Server Stream Recv:", res.Output)
    }
    return nil
}
```
启动服务端，客户端执行`serverStream(c, &pb.StreamRequest{Input: "something"})`，输出如下：
```
Server Stream Send: something
Server Stream Recv: index: 0, result: s
Server Stream Recv: index: 1, result: o
Server Stream Recv: index: 2, result: m
Server Stream Recv: index: 3, result: e
Server Stream Recv: index: 4, result: t
Server Stream Recv: index: 5, result: h
Server Stream Recv: index: 6, result: i
Server Stream Recv: index: 7, result: n
Server Stream Recv: index: 8, result: g
```

不难看出，客户端一次请求，服务端将数据多次返回，从而实现服务端流式响应

## 客户端流式请求

与服务端流式响应类似，只不过转变为服务端for循环调用`stream.Rece()`，接收客户端消息并阻塞，等客户端调用`stream.CloseAndRecv()`，关闭流的发送后进入阻塞监听，服务端调用`stream.SendAndClose()`，返回响应体并关闭流。此方式客户端只负责发送流的结束，服务端可以在中途结束整个流处理。

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
```

```go
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
客户端执行`clientStream(c, "something")`，输出如下：

```
Client Stream Send: s
Client Stream Send: o
Client Stream Send: m
Client Stream Send: e
Client Stream Send: t
Client Stream Send: h
Client Stream Send: i
Client Stream Send: n
Client Stream Send: g
Client Stream Recv: something
```

## 双向流式

由客户端发送流式请求，服务端通过流式响应，但具体的交互方式，根据编写的逻辑不同，效果也不同，类似于聊天室，开启聊天室后，怎么回复，回复多少，何时关闭，谁来关闭，根据适用场景而定。以下事例代码，客户端通过发送0-10，服务端累加返回。

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
```

```go
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
客户端执行`streaming(c)`，输出如下：

```
Streaming Send: 0
Streaming Recv: 0
Streaming Send: 1
Streaming Recv: 1
Streaming Send: 2
Streaming Recv: 3
Streaming Send: 3
Streaming Recv: 6
Streaming Send: 4
Streaming Recv: 10
.....
```

## 小结

本篇只是写了gRPC三种流式处理的简单demo，实际情况下，根据业务背景来选择该使用哪种流。例如双向流就类似于聊天室，或保持长连接类型，而传输数量量大时，可选择单向流，让接收方批量处理。

## 参考

- [gRPC那些事之Streaming](https://colobu.com/2017/04/06/dive-into-gRPC-streaming/)
- [带入gRPC：gRPC Streaming](https://segmentfault.com/a/1190000016503114)
