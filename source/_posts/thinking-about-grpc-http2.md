---
title: 思考gRPC ：为什么是HTTP/2
date: 2018-07-13 20:03:54
tags:
 - rpc
 - http2
 - grpc

categories:
 - 技术
---



## 背景

gRPC是google开源的高性能跨语言的RPC方案。gRPC的设计目标是在任何环境下运行，支持可插拔的负载均衡，跟踪，运行状况检查和身份验证。它不仅支持数据中心内部和跨数据中心的服务调用，它也适用于分布式计算的最后一公里，将设备，移动应用程序和浏览器连接到后端服务。

* https://grpc.io/
* https://github.com/grpc/grpc

### GRPC设计的动机和原则

* https://grpc.io/blog/principles

个人觉得官方的文章令人印象深刻的点：

* 内部有Stubby的框架，但是它不是基于任何一个标准的
* 支持任意环境使用，支持物联网、手机、浏览器
* 支持stream和流控


### HTTP/2是什么

在正式讨论gRPC为什么选择HTTP/2之前，我们先来简单了解下HTTP/2。 

HTTP/2可以简单用一个图片来介绍：

![HTTP/2](/img/http2.svg)

来自：https://hpbn.co/

可以看到：
* HTTP/1里的header对应HTTP/2里的 HEADERS frame
* HTTP/1里的payload对应HTTP/2里的 DATA frame


在Chrome浏览器里，打开`chrome://net-internals/#http2`，可以看到http2链接的信息。

![chrome-http2](/img/chrome-http2.png)

目前很多网站都已经跑在HTTP/2上了，包括alibaba。


### gRPC over HTTP/2

准确来说gRPC设计上是分层的，底层支持不同的协议，目前gRPC支持：

* [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
* [gRPC Web](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-WEB.md)

但是大多数情况下，讨论都是基于gRPC over HTTP2。

下面从一个真实的gRPC `SayHello`请求，查看它在HTTP/2上是怎样实现的。用wireshark抓包：

![wireshark-grpc](/img/wireshark-grpc.png)

可以看到下面这些Header：

* Header: :authority: localhost:50051
* Header: :path: /helloworld.Greeter/SayHello
* Header: :method: POST
* Header: :scheme: http
* Header: content-type: application/grpc
* Header: user-agent: grpc-java-netty/1.11.0

然后请求的参数在DATA frame里：

* GRPC Message: /helloworld.Greeter/SayHello, Request

简而言之，gGRPC把元数据放到HTTP/2 Headers里，请求参数序列化之后放到 DATA frame里。


### 基于HTTP/2 协议的优点

#### HTTP/2 是一个公开的标准

Google本身把这个事情想清楚了，它并没有把内部的Stubby开源，而是选择重新做。现在技术越来越开放，私有协议的空间越来越小。

#### HTTP/2 是一个经过实践检验的标准

HTTP/2是先有实践再有标准，这个很重要。很多不成功的标准都是先有一大堆厂商讨论出标准后有实现，导致混乱而不可用，比如CORBA。HTTP/2的前身是Google的[SPDY](https://en.wikipedia.org/wiki/SPDY)，没有Google的实践和推动，可能都不会有HTTP/2。

#### HTTP/2 天然支持物联网、手机、浏览器

实际上先用上HTTP/2的也是手机和手机浏览器。移动互联网推动了HTTP/2的发展和普及。

### 基于HTTP/2 多语言客户端实现容易

只讨论协议本身的实现，不考虑序列化。

* 每个流行的编程语言都会有成熟的HTTP/2 Client
* HTTP/2 Client是经过充分测试，可靠的
* 用Client发送HTTP/2请求的难度远低于用socket发送数据包/解析数据包

#### HTTP/2支持Stream和流控

在业界，有很多支持stream的方案，比如基于websocket的，或者[rsocket](https://github.com/rsocket/rsocket)。但是这些方案都不是通用的。

HTTP/2里的Stream还可以设置优先级，尽管在rpc里可能用的比较少，但是一些复杂的场景可能会用到。


#### 基于HTTP/2 在Gateway/Proxy很容易支持

* nginx对gRPC的支持：https://www.nginx.com/blog/nginx-1-13-10-grpc/
* envoy对gRPC的支持：https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/grpc#

#### HTTP/2 安全性有保证

* HTTP/2 天然支持SSL，当然gRPC可以跑在clear text协议（即不加密）上。 
* 很多私有协议的rpc可能自己包装了一层TLS支持，使用起来也非常复杂。开发者是否有足够的安全知识？使用者是否配置对了？运维者是否能正确理解？
* HTTP/2 在公有网络上的传输上有保证。比如这个[CRIME攻击](https://en.wikipedia.org/wiki/CRIME)，私有协议很难保证没有这样子的漏洞。

#### HTTP/2 鉴权成熟

* 从HTTP/1发展起来的鉴权系统已经很成熟了，可以无缝用在HTTP/2上
* 可以从前端到后端完全打通的鉴权，不需要做任何转换适配

比如传统的rpc dubbo，需要写一个dubbo filter，还要考虑把鉴权相关的信息通过thread local传递进去。rpc协议本身也需要支持。总之，非常复杂。实际上绝大部分公司里的rpc都是没有鉴权的，可以随便调。


### 基于HTTP/2 的缺点

* rpc的元数据的传输不够高效

    尽管HPAC可以压缩HTTP Header，但是对于rpc来说，确定一个函数调用，可以简化为一个int，只要两端去协商过一次，后面直接查表就可以了，不需要像HPAC那样编码解码。
    可以考虑专门对gRPC做一个优化过的HTTP/2解析器，减少一些通用的处理，感觉可以提升性能。

* HTTP/2 里一次gRPC调用需要解码两次

    一次是HEADERS frame，一次是DATA frame。

* HTTP/2 标准本身是只有一个TCP连接，但是实际在gRPC里是会有多个TCP连接，使用时需要注意。


gRPC选择基于HTTP/2，那么它的性能肯定不会是最顶尖的。但是对于rpc来说中庸的qps可以接受，通用和兼容性才是最重要的事情。

* 官方的benchmark：https://grpc.io/docs/guides/benchmarking.html
* https://github.com/hank-whu/rpc-benchmark 

### Google制定标准的能力

近10年来，Google制定标准的能力越来越强。下面列举一些标准：

* HTTP/2
* WebP图片格式
* WebRTC 网页即时通信
* VP9/AV1 视频编码标准
* Service Worker/PWA

当然google也并不都会成功，很多事情它想推也失败了，比如Chrome的Native Client。

**gRPC目前是k8s生态里的事实标准。 gRPC是否会成为更多地方，更大领域的RPC标准？**

### 为什么会出现gRPC

准确来说为什么会出现基于HTTP/2的RPC？

个人认为一个重要的原因是，在Cloud Native的潮流下，开放互通的需求必然会产生基于HTTP/2的RPC。即使没有gRPC，也会有其它基于HTTP/2的RPC。

gRPC在Google的内部也是先用在Google Cloud Platform和公开的API上：https://opensource.google.com/projects/grpc

尽管gRPC它可能替换不了内部的RPC实现，但是在开放互通的时代，不止在k8s上，gRPC会有越来越多的舞台可以施展。


### 链接

* https://hpbn.co/
* https://grpc.io/blog/loadbalancing
* https://http2.github.io/faq

