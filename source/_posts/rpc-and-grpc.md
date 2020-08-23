---
title: 初识gRPC
date: 2020-08-23 23:56:46
tags:
  - gRPC
categories:
  - gRPC
---
## 前言

最近我司技术栈新增加了gRPC，小弟有幸使用gRPC实现了几个业务需求，本篇文章记录对RPC以及gRPC的一些浅显认知。

## RPC

RPC，全称`remote procedure call`，即远程过程调用，也是一种客户端-服务器（client/server）模型，通俗讲RPC就是客户端像调用本地方法一样调用了远端服务中的方法，而无需关心具体调用细节。

这里先看看本地是如何调用函数的，伪代码如下：

```python
def add(x, y):
    return x + y

a, b = 5, 10
result = add(a, b)
```

执行上述代码，大致经过以下步骤

1. 定义a,b变量并赋值
2. 执行add函数，将a,b值，赋值给x,y
3. 执行x+y
4. 退出add函数，将结果赋值给result

以上4步就是本地调用的过程。在不考虑调用函数逻辑情况下，调用RPC方法体验与本地函数调用基本一致。

不过因为RPC涉及网络调用，服务两端并不存在一个内存空间内，这样就会带来一些新问题

**1.服务之间方法如何映射？**

本地调用通过函数指针指定调用的函数，而RPC调用为了确保执行正确的函数，则需要客户端与服务端有一致函数映射，确保服务端执行的是客户端调用的，简单实现可以用过双方维护一套映射表，一般通过动态代理实现。

**2.服务之间传输的信息结构？**

本地调用只需将参数压入栈中，函数从栈中读取即可。而RPC调用，因为不在相同的内存空间，无法通过内存传递参数，并且需要通过网络传输，则需要一套序列化/反序列化机制，确保服务端将收到的 客户端的序列化后的参数 反序列化后 与 实际传参 意义一致。

**3.服务之间如何通信？**

客户端需要通过网络传输，将函数映射以及序列化后的内容发送给服务端，而服务端则需要返回内容给客户端。中间则需要选择传输协议，可以是传输层协议，也可以是应用层协议，如gRPC就使用HTTP/2。

解决了上述三个问题，也就实现了一个基础的RPC框架。

### RPC调用流程

下图为一个完成的RPC过程：

![image](https://static001.geekbang.org/resource/image/82/59/826a6da653c4093f3dc3f0a833915259.jpg)

> 笼统的讲，封装好的HTTP接口形式也可算作RPC的一种

## gRPC

gRPC是google与2015年开源的RPC框架，相比于其他RPC框架，最显著的三大特点：

1. `ProtoBuf`作为默认IDL
2. 基于`HTTP/2`协议，具体HTTP/2实现可移步[lxkaka](http://lxkaka.wang/http2/)
3. 支持多种开发语言

![gRPC交互流程](https://colobu.com/2017/04/06/dive-into-gRPC-streaming/gRPC.png)

### gRPC vs REST

**高性能**

基于HTTP/2的多路复用，减少TCP连接，首部压缩等特性以及ProtoBuf高效的二进制序列化，更高效的传输消息，节省带宽，从而提高吞吐量。

**流处理**

HTTP/2为实时通信流提供了基础。除了简单的RPC，gRPC中支持三种流组合：

- 服务器流式响应
- 客户端流式发送
- 双向流

**开发流程简单**

相比于RESTful接口，无需思考请求方式，URL路径等规范，定义好proto请求响应结构，生成对应pb文件即可。

**支持多语言**

只需编写一份proto，通过命令行生成对应语言的Stub代码即可调用，而RESTful接口，每个调用方都需要实现一套基础调用类。

### 开发流程

体验下来，gRPC大致的开发流程如下：

1. 定义proto文件，即定义Request和Response结构，以及包含多个方法的服务Service
2. 通过protoc工具生成对应语言的Stub
3. server实现proto中定义的接口，编写逻辑代码
4. client通过生成的Stub调用server

### 示例代码

Go编写一个获取服务器时间的rpc接口，示例代码如下：

**time.proto**

```protobuf
syntax = "proto3";

package proto;

option go_package = "base;base";

service BaseService {
    rpc GetTime (TimeRequest) returns (TimeResponse){}
}

message TimeRequest {}

message TimeResponse {
  string time = 1;
}
```

通过protoc生成服务端以及客户端的Stub代码

```
protoc --go_out=. --go-grpc_out=.  time.proto
```

**server.go**

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

func (s *service) GetTime(ctx context.Context, in *pb.TimeRequest) (*pb.TimeResponse, error) {
	now := time.Now().Format("2006-01-02 15:04:05")
	return &pb.TimeResponse{Time: now}, nil
}
```

**client.go**

```go
package main

import (
	"context"
	"fmt"
	"time"

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

	getTime(c)
}

func getTime(client pb.BaseServiceClient) error {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, _ := client.GetTime(ctx, &pb.TimeRequest{})
	fmt.Println("Time:", r.Time)
	return nil
}
```

## 小结

对于简单的需求实现，gRPC很容易上手，编写protobuf并不需要额外去了解什么语法，类似interface。而如果要实现gRPC高可用，还有很多路要走。



###  参考

- [gRPC官方文档](https://grpc.io/docs/what-is-grpc/introduction/)
- [通俗解释一下什么是RPC](https://www.zhihu.com/question/25536695)
