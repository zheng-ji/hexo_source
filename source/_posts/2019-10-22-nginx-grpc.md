---
title: Nginx 支持 GRPC 负载均衡
date: 2019-10-22 12:11:42
tags:  nginx grpc
categories: System
---

接触 grpc，从它的 [example-GreetHelloWorld](https://github.com/grpc/grpc-go/tree/master/examples/helloworld) 开始, client 指定了 Server 监听的IP和端口，然后直接访问.就完成了一次 RPC 请求。

那时候我就在想，这里是否能指定代理转发，做负载均衡呢。比如 http访问下，我们会用 proxy_pass + upstream 来代理服务端，这样来实现服务端的负载均衡，以及灰度发布。

今日偶然发现 nginx 1.13 已经 支持 grpc 协议， 实操上我们可以像使用 proxy_pass 一样使用 grpc_pass 

```shell
server {
    listen 80 http2;
    access_log logs/access.log main;
    location / {
        // Your Server Address
        grpc_pass grpc://localhost:50051;
    }
}
```
