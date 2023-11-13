---
title: GRPC Gateway 入门
date: 2021-09-28 12:11:42
tags:  grpc
categories: grpc
---

在某些情况下，即使我们写了 gRPC 服务，但我们仍然想提供传统的 HTTP/JSON API。但是仅仅为了公开 HTTP/JSON API 而编写另一个服务有点不友好。
有什么方法可以只编写一次代码，却可以同时在 gRPC 和 HTTP/JSON 中提供 API？ 

gRPC-gateway 可以帮我们做到，它读取 protobuf service 定义并生成反向代理服务器( reverse-proxy server) ，根据服务定义中的 `google.api.http annotations` 将 RESTful HTTP API 转换为 gRPC。

![](/images/grpc/grpc-gateway.png)

### 安装

在这之前需要先安装好 protoc, 

```
sudo apt-get install protoc
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.26
$ go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.1
$ go install github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway@v1.16.0 
$ export PATH="$PATH:$(go env GOPATH)/bin"
```

### 编写 ProtoBuf

```
|~proto/
| |~api/
| | |-hello.proto
| `~google/
|   `~api/
|     |-annotations.proto
|     `-http.proto
```

在根目录执行 `go mod init grpc-gateway-exp`, 创建 `proto` 目录，从 `https://github.com/googleapis/googleapis/tree/master/google/api` 下载 `annotation.proto` 和 `http.proto` 放置于 `proto/google/api` 目录下, 主要是用于 http 服务的注解，比如:

```
syntax = "proto3";
package grpc_gateway_exp;
option go_package = "grpc_gateway_exp";
import "google/api/annotations.proto"; // 会在当前根目录开始搜索

service HelloService
{
    rpc Hello(HelloMessage) returns (HelloResponse)
    {
        # 魔法发生的地方
        option (google.api.http) = {
            get: "/hello"
        };
    }
}

message HelloMessage 
{
    string message = 1;
}

message HelloResponse 
{
    string result = 1;
}
```

### 借助 [buf](https://github.com/bufbuild/buf) 生成 ProtoBuf 所需代码

安装 `buf` 的命令

```
BIN="/usr/local/bin" && \
VERSION="1.0.0-rc2" && \
BINARY_NAME="buf" && \
curl -sSL \
     "https://github.com/bufbuild/buf/releases/download/v${VERSION}/${BINARY_NAME}-$(uname -s)-$(uname -m)" \
     -o "${BIN}/${BINARY_NAME}" && \
chmod +x "${BIN}/${BINARY_NAME}"
```

在项目根目录创建 `buf.yaml`，`buf.gen.yaml`

```buf.yaml
version: v1beta1
build:
  roots:
    - proto # proto的目录
```

```buf.gen.yaml
version: v1beta1
plugins:
  - name: go
    out: proto
    opt: paths=source_relative
  - name: go-grpc   # go-grpc plugin
    out: proto
    opt: paths=source_relative,require_unimplemented_servers=false # 相对路径引用
  - name: grpc-gateway # grpc-gateway plugin
    out: proto
    opt: paths=source_relative # 相对路径引用
```

执行 `buf generate` 之后就生成了。

### Go Mod 的使用

项目运用了 `Go Module` , 以期读者何时何地的下载，都能直接使用。
项目分了子 package, 诸如 server, service, proto。
由于在根目录执行了 go mod init grpc-gateway-exp 所以对子 package 的引用, 可以用如下的写法。这个经典的写法应该引起注意。

```
import (
	pb "grpc-gateway-exp/proto/api" // 看这里
	"grpc-gateway-exp/service" // 看这里
)
```

###  写 grpc-gateway

```
package server

import (
	"context"
	"log"
	"net/http"
	"github.com/grpc-ecosystem/grpc-gateway/runtime"
	"google.golang.org/grpc"
	pb "grpc-gateway-exp/proto/api"
)

func StartGwServer() {
	conn, err := grpc.DialContext(
		context.Background(),
		"0.0.0.0:9090", // 背后的RPC Server
		grpc.WithBlock(),
		grpc.WithInsecure(),
	)
	if err != nil {
		log.Fatalln("Failed to dial server: ", err)
	}
	mux := runtime.NewServeMux()
	err = pb.RegisterHelloServiceHandler(context.Background(), mux, conn)

	if err != nil {
		log.Fatalln("Failed to register gateway: ", err)
	}

	server := &http.Server{
		Addr:    ":8090",
		Handler: mux,
	}

	log.Println("Start gRPC Gateway Server on http://0.0.0.0:8090")
	err = server.ListenAndServe()
	if err != nil {
		log.Fatalln("Start Gateway Server failed: ", err)
	}
}
```

* 效果
```
curl localhost:8090/hello?message=world
$ hello world
```


上述完整代码 [Link](https://github.com/zheng-ji/grpc-gateway-exp)






