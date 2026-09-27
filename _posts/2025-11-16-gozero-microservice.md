---
layout: post
title: go zero 介绍
date: 2025-11-16 +0800
tags: ["Go"]
---

> `go zero`是一个集成了各种工程实践（包含`web`和`rpc`）的框架，有如下主要特点：1、强大的工具支持，尽可能少的代码编写；2、极简的接口，完全兼容net/http;
> 3、支持中间件，高性能，面向故障编程，弹性设计；4、内建服务发现，智能负载均衡；5、高并发支撑，稳定保障每天的流量洪峰；

<!-- more -->
### 为什么说做好微服务很难？
* 基础功能层面：
1、并发控制&限流，避免服务被突发流量击垮；2、服务注册与服务发现，确保能够动态侦测增减的节点；3、负载均衡，需要根据节点承受能力分发流量；4、超时控制，避免对已超时请求做无用功；5、熔断设计，快速失败，保障故障节点的恢复能力；
* 高阶层面功能：
1、请求认证，确保每个用户只能访问自己的数据；2、链路追踪，用于理解整个系统和快速定位特定请求的问题；3、日志，用于数据收集和问题定位；4、可观测性，没有度量就没有优化；

### goctl的用法

1. 使用 `goctl` 生成 `dockerfile`，本地构建镜像，然后启动服务测试。

```shell
# go-zero/examples目录下执行`goctl apin ew hello`，创建名为`hello`的api
goctl api new hello
```
语句执行完成后，会生成 `hello` 服务的目录，进入`hello`目录，然后执行`goctl docker`命令，会在`hello`目录下生成`dockerfile`。
```shell
goctl docker -go hello.go
# 回到bellagio目录，执行docker build构建镜像
docker build -t hello:v1 -f go-zero/examples/hello/Dockerfile . 
# 启动hello服务，暴露8888的端口
docker run --rm -it -p 8888:8888 hello:v1 
# 使用curl请求接口，可正常返回message信息
curl -i http://localhost:8888/from/me  
{"message":"from docker, hello, me"}%
```
这里需要注意，要修改dockerfile中的内容，`RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories`将软件源替换为阿里云镜像，加速软件包下载，否则在构建时，会提示`build`阶段失败。

参考： 
- docker镜像构建与服务部署：https://cheney.app/go-zero-docker-build-deployment-guide
- docker build报错，需将buildkit的值由true改为false: https://blog.csdn.net/qq_40671063/article/details/129097796
2. 使用 `goctl` 生成 `api` 和 `proto` 代码

```shell
# 在gozero/examples/shorturl/api目录下，编写好shorturl.api文件后，用goctl生成api gateway代码
goctl api go -api shorturl.api -dir .

# 在gozero/examples/shorturl/rpc目录下，编写好transform.proto后，用goctl生成proto文件模版
goctl rpc -o transform.proto
# 用goctl生成rpc代码，在rpc/transform目录下执行命令
goctl rpc protoc transform.proto --go_out=./pb --go-grpc_out=./pb --zrpc_out=.

# 在rpc/transform/model目录下创建shorturl.sql，使用goctl生成crud代码，其中-c表示使用redis cache
goctl model mysql ddl -c -src shorturl.sql -dir .
```
`goctl` 命令执行完成后，最终生成的目录结构如 `go-zero/examples/shorturl` 所示。

### 扩展阅读，`Go Zero`系列文章：
- [go zero微服务框架介绍](https://mp.weixin.qq.com/s/gszj3-fwfcof5Tt2Th4dFA)
- [微服务注册与发现的原理和实现](https://mp.weixin.qq.com/s/-WaWJaM_ePEQOf7ExNJe7w)
- [基于gRPC的注册发现与负载均衡原理](https://mp.weixin.qq.com/s/olPGfrFczo22rhpPLBmpNw)
- [如何让gRPC具备微服务治理能力](https://mp.weixin.qq.com/s/T88VCqsmSrUcm0R2iUelHQ)
- [通过SingleFlight模式学习Go并发编程](https://mp.weixin.qq.com/s/il_JuDrr2zsmjjGiJAp-fw)
- [服务发现原理分析与源码解读](https://mp.weixin.qq.com/s/E77rhCKqaRi4JW63UKA4qA)
- [负载均衡原理分析与源码解读](https://mp.weixin.qq.com/s/FwBlCB0RKUOeq6PiKr2GQw)
- [熔断原理分析与源码解读](https://mp.weixin.qq.com/s/4JLDjEVoYnceKcbBH4zm5w)