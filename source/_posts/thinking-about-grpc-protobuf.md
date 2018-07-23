---
title: 思考gRPC ：为什么是protobuf
date: 2018-07-17 20:03:54
tags:
 - rpc
 - http2
 - grpc
 - protobuf
 - security

categories:
 - 技术
---


## 背景

谈到RPC，就避免不了序列化的话题。

gRPC默认的序列化方式是protobuf，原因很简单，因为两者都是google发明的，哈哈。

在当初Google开源protobuf时，很多人就期待是否能把RPC的实现也一起开源出来。没想到最终出来的是gRPC，终于补全了这一块。

## 跨语言的序列化方案

事实上的跨语言序列化方案只有三个： protobuf, thrift, json。

* json体积太大，并且缺少类型信息，实际上只用在RESTful接口上，并没有看到RPC框架会默认选json做序列化的。

国内一些大公司的使用情况：

* protobuf ，腾迅，百度等

* thrift，小米，美团等

* hessian， 阿里用的是自己维护的版本，有js/cpp的实现，因为阿里主用java，更多是历史原因。

### 序列化里的类型信息

序列化就是把对象转换为二进制数据，反序列化就把二进制数据转换为对象。

各种序列化库层出不穷，其中有一个重要的区别：**类型信息存放在哪**？ 

可以分为三种：

1. 不保存类型信息

    典型的是各种json序列化库，优点是灵活，缺点是使用的双方都要知道类型是什么。当然有一些json库会提供一些扩展，偷偷把类型信息插入到json里。

1. 类型信息保存到序列化结果里

    比如java自带的序列化，hessian等。缺点是类型信息冗余。比如RPC里每一个request都要带上类型。因此有一种常见的RPC优化手段就是两端协商之后，后续的请求不需要再带上类型信息。

1. 在生成代码里带上类型信息

    通常是在IDL文件里写好package和类名，生成的代码直接就有了类型信息。比如protobuf, thrift。缺点是需要生成代码，双方都要知道IDL文件。


类型信息看起来是一个小事，但在安全上却会出大问题，后面会讨论。

## 实际使用中序列化有哪些问题

这里讨论的是没有IDL定义的序列化方案，比如java自带的序列化，hessian, 各种json库。

* 大小莫名增加，比如用户不小心向map里put了大对象。
* 对象之间互相引用，用户根本不清楚序列化到底会产生什么结果，可能新加一个field就不小心被序列化了
* enum类新增加的不能识别，当两端的类版本不一致时就会出错
* 哪些字段应该跳过序列化 ，不同的库都有不同的 @Ignore ，没有通用的方案
* 很容易把一些奇怪的类传过来，然后对端报ClassNotFoundException
* 新版本jdk新增加的类不支持，需要序列化库不断升级，如果没人维护就悲剧了
* 库本身的代码质量不高，或者API设计不好容易出错，比如[kryo](https://blog.csdn.net/hengyunabc/article/details/7764509)


### gRPC是protobuf的一个插件

以gRPC官方的Demo为例：

```java
package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

可以看到rpc的定义也是写在proto文件里的。实际上gRPC是protobuf的一个扩展，通过扩展生成gRPC相关的代码。

## protobuf并不是完美解决方案

在protobuf出来以后，也不断出现新的方案。比如 

* https://github.com/capnproto/capnproto
* https://github.com/google/flatbuffers
* https://avro.apache.org/


protobuf的一些缺点：

* 缺少map/set的支持(proto3支持map)
* [Varint](https://developers.google.com/protocol-buffers/docs/encoding)编码会消耗CPU
* 会影响CPU缓存，比如比较大的int32从4字节用Varint表示是5字节就不对齐了
* 解码时要复制一份内存，不能做原地内存引用的优化

    protobuf在google 2008年公开的，内部使用自然更早。当时带宽还比较昂贵，现在人们对速度的关注胜过带宽了。


protobuf需要生成代码的确有点麻烦，所以会有基于java annotation的方案：

* https://github.com/protostuff/protostuff

同样thrift有：

* https://github.com/facebookarchive/swift

## 序列化库的速度问题

总有序列化库跳出来说自己速度最快，其实很多时候猫腻很多。事有反常必有妖。

常见的加快速度的手段有：

* threadlocal的byte array

    当序列化一个大对象后，threadlocal的byte array增大，然后不能及时释放。如果线程池越大，则占用的内存会越多。fastjson采用一种动态缩小的处理办法，但不能从根本解决这个问题。

* 用asm的方式生成代码，避免反射调用getter/setter

    这样会导致库代码复杂，容易有bug，并且会占用内存。

* 循环引用用ID标识对象

    kryo要求注册类型的顺序是统一的，因为它要为类型分配ID，然后在处理循环引用时，把同样的对象直接用ID来标识，这样子可以大大减少体积。
    但是用户在使用时，调用代码的顺序可能是不确定的，注册上去的ID也可能不一样，那么反序列化就会有问题。
    kryo的API还不是线程安全的，很容易踩坑。


在benchmark里protobuf的速度在前列，并不是最快。但是protobuf用生成代码的方式保证了内存占用，时间占用不会出问题。

## 序列化被人忽视的安全性问题

### 序列化漏洞危害很大

1. 序列化漏洞通常比较严重，容易造成任意代码执行
1. 序列化漏洞在很多语言里都会有，比如Python Pickle序列化漏洞。

很多程序员不理解为什么反序列化可以造成任意代码执行。

反序列化漏洞到底是怎么工作的呢？很难直接描述清楚，这些漏洞都有很精巧的设计，把多个地方的代码串联起来。可以参考这个demo，跑起来调试下就可以有直观的印象：

* https://github.com/hengyunabc/dubbo-apache-commons-collections-bug


这里有两个生成java序列化漏洞代码的工具：

* https://github.com/frohoff/ysoserial
* https://github.com/mbechler/marshalsec


### 常见的库怎样防止反序列化漏洞

下面来看下常见的序列化方案是怎么防止反序列化漏洞的：

1. Java Serialization

    * jdk里增加了一个filter机制 http://openjdk.java.net/jeps/290 ，这个一开始是出现在jdk9上的，后面移值回jdk6/7/8上，如果安装的jdk版本是比较新的，可以找到相关的类
    * Oracle打算废除java序列化：https://www.infoworld.com/article/3275924/java/oracle-plans-to-dump-risky-java-serialization.html 

1. jackson-databind

    * jackson-databind里是过滤掉一些已知的类，参见[SubTypeValidator.java](https://github.com/FasterXML/jackson-databind/blob/jackson-databind-2.9.6/src/main/java/com/fasterxml/jackson/databind/jsontype/impl/SubTypeValidator.java)
    * jackson-databind的[CVE issue列表](https://github.com/FasterXML/jackson-databind/issues?q=is%3Aissue+label%3ACVE+is%3Aclosed)

1. fastjson

    * fastjson通过一个`denyList`来过滤掉一些危险类的package，参见[ParserConfig.java](https://github.com/alibaba/fastjson/blob/1.2.7.sec01/src/main/java/com/alibaba/fastjson/parser/ParserConfig.java#L169)
    * fastjson在新版本里`denyList`改为通过hashcode来隐藏掉package信息，但通过这个[`DenyTest5`](https://github.com/alibaba/fastjson/blob/1.2.47/src/test/java/com/alibaba/json/bvt/parser/deser/deny/DenyTest5.java)可以知道还是过滤掉常见危险类的package
    * fastjson在新版本里默认把`autoType`的功能禁止掉了

所以总结下来，要么白名单，要么黑名单。当然黑名单机制不能及时更新，业务方得不断升jar包，非常蛋疼。白名单是比较彻底的解决方案。

### 为什么protobuf没有序列化漏洞

**这些序列化漏洞的根本原因是：没有控制序列化的类型范围**

为什么在protobuf里并没有这些反序列化问题？

* protobuf在IDL里定义好了package范围
* protobuf的代码都是自动生成的，怎么处理二进制数据都是固定的

protobuf把一切都框住了，少了灵活性，自然就少漏洞。

## 总结

* 应该重视反序列化漏洞，毕竟Oracle都不得不考虑把java序列化废弃了
* 序列化漏洞的根本原因是：没有控制序列化的类型范围
* 防止序列化漏洞，最好是使用白名单
* protobuf通过IDL生成代码，严格控制了类型范围
* protobuf不是完美的方案，但是作为跨语言的序列化事实方案之一，IDL生成代码比较麻烦也不是啥大问题

## 链接 
* https://github.com/protostuff/protostuff
* https://github.com/facebookarchive/swift
* http://openjdk.java.net/jeps/290
* https://www.infoworld.com/article/3275924/java/oracle-plans-to-dump-risky-java-serialization.html 
