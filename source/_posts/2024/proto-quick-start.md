---
layout: post
title: protoc quick start - golang
# subtitle: proto-quick-start
comments: true
tags: ["Protocol Buffers", "protoc", "grpc", "go"]
# header-img: 2024/proto-quick-start/banner.jpg
category: ["tech", "grpc", "Protocol Buffers"]
date: 2024-11-20 23:51:17
updated: 2024-11-20 23:51:17
---

## 官方网站

<https://protobuf.dev/>
<https://github.com/protocolbuffers/protobuf>
<https://grpc.io/>
<https://github.com/grpc/grpc>

## 安装

<https://github.com/protocolbuffers/protobuf/releases>

下载对应系统的 zip 包，解压就可以得到编译后的 protoc 二进制文件。
在类 unix 系统下，可以将 include 中的 proto 文件拷贝到 '/usr/local/include/' 下，方便引用官方支持的类型。

```bash
mkdir ~/temp && cd ~/temp
curl -OL https://github.com/google/protobuf/releases/download/v3.20.3/protoc-3.20.3-linux-x86_64.zip \
    && mkdir protoc3 \
    && unzip protoc-3.20.3-linux-x86_64.zip -d protoc3 \
    && mv protoc3/bin/* /usr/local/bin/ \
    && mv protoc3/include/* /usr/local/include/
```

<https://github.com/protocolbuffers/protobuf-go/tree/master/cmd/protoc-gen-go>
<https://github.com/grpc/grpc-go/tree/master/cmd/protoc-gen-go-grpc>
<https://github.com/grpc-ecosystem/grpc-gateway/tree/main/protoc-gen-grpc-gateway>
<https://github.com/grpc-ecosystem/grpc-gateway/tree/main/protoc-gen-openapiv2>

我们使用 grpc-gateway 作为 demo 演示，因此需要安装 go 语言插件以及 grpc, grpc-gateway 插件。openapiv2 用于生成 swagger 文档。

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest
```

### xxx_out=xxx 和插件 protoc-gen-xxx 的关系

对于 protoc 插件，命名为 `protoc-gen-xxx`，与 protoc 的参数 `--xxx_out` 所对应。
举例来说，protoc 在生成 go 语言模版代码时，使用的是插件 `protoc-gen-go` 来生成，所以命令就是 `protoc -I=. --go_out=. demo.proto`

## 简单使用

语法参考[官网](https://protobuf.dev/programming-guides/proto3/)，Protocol Buffers 最基础的功能就是序列化和反序列化。

```proto demo.proto
syntax = 'proto3';

option go_package='./demo';

message HelloRequest {
	string name = 1;
}

```

```bash
protoc -I=. --go_out=. demo.proto
```

```go main.go
package main

import (
	"fmt"
	"protoc-demo/demo"

	"google.golang.org/protobuf/proto"
)

func main() {
	b, err := proto.Marshal(&demo.HelloRequest{
		Name: "Foo",
	})
	if err != nil {
		panic(err)
	}

	fmt.Printf("125ns encoded into %d bytes of Protobuf wire format:\n% x\n", len(b), b)

	var hello demo.HelloRequest
	if err := proto.Unmarshal(b, &hello); err != nil {
		panic(err)
	}

	fmt.Printf("Protobuf wire format decoded to duration %v\n", hello.Name)
}

```

```bash
mkdir temp && cd temp
go mod init protoc-demo
# 将 demo.proto, main.go 置入当前目录
protoc -I=. --go_out=. demo.proto
go mod tidy
go run main.go

```

## 使用 grpc-gateway 创建 restful api

使用 grpc-gateway 需要添加 proto 扩展 `google/api/annotations.proto`。

下载 google/api/http.proto, 并放到项目的 google/api 路径下。
<https://github.com/googleapis/googleapis/blob/master/google/api/annotations.proto>
<https://github.com/googleapis/googleapis/blob/master/google/api/http.proto>

```proto demo.proto
syntax = 'proto3';

option go_package='./demo';

import "google/api/annotations.proto";

service HelloService {
    rpc Hello(HelloRequest) returns (HelloResponse) {
        option (google.api.http) = {
            get: "/api/v1/hello/{name}"
        };
    }
}

message HelloRequest {
    string name = 1;
}

message HelloResponse {
    string msg = 1;
}

```

```bash
protoc -I=. --go_out=. --go-grpc_out=. --grpc-gateway_out=. demo.proto
```

```go main.go
package main

import (
	"context"
	"log"
	"net"
	"net/http"
	"os"
	"os/signal"
	"protoc-demo/demo"
	"syscall"

	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

var (
	rpcServerPort = ":9001"
	gatewayPort   = ":9000"
)

type server struct {
	demo.UnimplementedHelloServiceServer
}

func (s *server) Hello(ctx context.Context, req *demo.HelloRequest) (*demo.HelloResponse, error) {
	log.Printf("Handle HelloService Hello() request name: %s \n", req.Name)
	return &demo.HelloResponse{Msg: "Hello " + req.GetName()}, nil
}

func main() {
	go runRpcServer()
	go runGateway()

	osExitSignal := make(chan os.Signal, 1)
	signal.Notify(osExitSignal, os.Interrupt, syscall.SIGTERM)
	sig := <-osExitSignal
	log.Printf("OS initiated shutdown: %v \n", sig)
}

func runRpcServer() {
	lis, err := net.Listen("tcp", rpcServerPort)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	s := grpc.NewServer()
	demo.RegisterHelloServiceServer(s, &server{})

	log.Printf("Rpc Server Start on: %s \n", rpcServerPort)
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

func runGateway() {
	ctx := context.Background()
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	mux := runtime.NewServeMux()
	opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
	err := demo.RegisterHelloServiceHandlerFromEndpoint(ctx, mux, rpcServerPort, opts)
	if err != nil {
		log.Fatalf("failed to RegisterHelloServiceHandlerFromEndpoint: %v", err)
	}

	log.Printf("gateway Server Start on: %s \n", gatewayPort)

	err = http.ListenAndServe(gatewayPort, mux)
	if err != nil {
		log.Fatalf("failed to ListenAndServe: %v", err)
	}
}

```

启动之后，执行 `curl http://localhost:9000/api/v1/hello/world!` 可以得到 Response, 则 gateway 执行成功。
