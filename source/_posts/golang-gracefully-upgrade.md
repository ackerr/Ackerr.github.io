---
title: Golang中实现平滑发版 
date: 2020-10-31 21:48:14
tags:
    - go
---

关于平滑发版，[紫月](https://github.com/LKI)写过一篇博客《[软件工程实践之平滑发版](https://liriansu.com/posts/2019-12-09-gracefully-upgrade/)》，其中讲解了为什么需要平滑发版以及服务中如何实现。总结下来，平滑发版中会经历的几个步骤：

1. 启动新进程，旧进程停止监听请求，新请求由新进程监听
2. 旧进程等待已连接的请求结束
3. 待请求全部执行完毕或超过超时时间，关闭进程

而进程什么时候停止监听，什么时候关闭进程，则可以通过Signal来实现。
以k8s为例，在执行`kubectl rollout restart`后，

1. 旧pod会接收到SIGTERM信号，pod status变为Terminating
2. 旧pod等待处理的中的请求结束
3. 待处理中的请求结束或超过预设的`terminationGracePeriodSeconds`时间
4. 旧pod接收到SIGKILL信号后被移除

一般来说，通过守护进程进行部署的方式，只需要正确处理Signal，即可实现服务业务平滑发版，例如最近整的项目中，由四部分组成：

- [gin](https://github.com/gin-gonic/gin)启动的web服务
- [machinery](https://github.com/RichardKnop/machinery)启动的异步任务
- [cron](https://github.com/robfig/cron)启动的定时任务
- [grpc-go](https://github.com/grpc/grpc-go)启动的grpc服务

其中machinery本身就实现了gracefully shutdown，调用api即可，后文将依次介绍其他三个部分如何实现平滑发版。

## gin

Gin官方提供了[gracefully shutdown的例子](https://github.com/gin-gonic/examples/blob/master/graceful-shutdown/graceful-shutdown/server.go)，代码如下

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gin-gonic/gin"
)

func main() {
    router := gin.Default()
    router.GET("/", func(c *gin.Context) {
        time.Sleep(5 * time.Second)
        c.String(http.StatusOK, "Hello World")
    })

    srv := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %s\n", err)
        }
    }()

    quit := make(chan os.Signal)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Println("Shutting down server...")

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }
    log.Println("Server exiting")
}
```

例子中通过主goroutine通过`signal.Notify()`监听， `<-quit`进行阻塞，另外一个goroutine启动http server，当接收到SIGINT或SIGTERM信号后，http.ListenAndServe()返回http.ErrServerClosed，此时httpserver不再接受新请求，并创建一个超时时间为5s的context，通过http.Server的[Shutdown](https://golang.org/pkg/net/http/#Server.Shutdown)，并阻塞等待连接释放或context超时。

## grpc-go

grpc-go中提供了`GracefulStop`，我们只需要监听Signal，接受到后，由GracefulStop来处理连接中的请求

```go
package main

import (
    "log"
    "net"
    "os"
    "os/signal"
    "syscall"

    "google.golang.org/grpc"
)

func main() {
    listen, err := net.Listen("tcp", ":8080")
    if err != nil {
        panic(err)
    }
    server := grpc.NewServer()
    // something...
    
    go func() {
        if err = server.Serve(listen); err != nil {
              panic(err)
        }
    }()
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Println("Shutting down gRPC server...")
    server.GracefulStop()
    log.Println("gRPC server exiting...")
}
```

## cron

robfig/cron包中虽然没提供关于gracefully shutdown的内容，至少提供了Start以及Stop方法，按照前几个实现的套路，我们可以自己动手实现一下gracefully shutdown。因为cron.Start本身会新起一个的goroutine启动server，所以我们在主goroutine中监听signal，在收到Signal后，执行cron.Stop()，而cron.Stop()会返回一个context，后续通过select监听context结束或者通过time.After等待超时，待当前所有定时任务结束后，进程结束。

```go
package main

import (
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    cronjob := cron.New()
    cronjob.AddFunc("@every 5s", func() { time.Sleep(10 * time.Second) })
    cronjob.Start()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    ctx := cronjob.Stop()
    log.Println("Shutting down cron...")
    select {
    case <-time.After(10 * time.Second):
        log.Fatal("Cron forced to shutdown...")
    case <-ctx.Done():
        log.Println("Cron exiting...")
    }
}
```

## 小结

本文只介绍了golang中如何实现平滑发版，一般框架都会给出关于gracefully shutdown的api，没有也没关系，套路就是正常启动server后，通过监听Signal并阻塞主goroutine，待接收到SIGNTERM等信号后，通过time.After或context.WithTimeout，作为超时处理，并等待进行中的goroutine全部结束。有兴趣可以查看各个类库中如何等待处理进行中的连接。

