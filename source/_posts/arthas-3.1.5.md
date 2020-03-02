---
title: Alibaba Arthas 3.1.5版本支持火焰图，快速定位应用热点
date: 2019-11-26 20:58:28
tags:
 - arthas
 - java
 - profiler


categories:
  - 技术

---


![Arthas](https://alibaba.github.io/arthas/_images/arthas.png)

`Arthas`是Alibaba开源的Java诊断工具，深受开发者喜爱。

* Github： https://github.com/alibaba/arthas
* 文档：https://alibaba.github.io/arthas

Arthas 3.1.5版本带来下面全新的特性：

* 开箱即用的Profiler/火焰图功能
* grep命令支持更丰富的选项
* monitor/tt/trace等命令提供更精确的时间统计
* telnet/http协议共用3658端口

## Profiler/Flame Graph/火焰图

火焰图的威名相信大家都有所耳闻，但可能因为使用比较复杂，所以望而止步。

在新版本的Arthas里集成了[async-profiler](https://github.com/jvm-profiling-tools/async-profiler)，使用`profiler`命令就可以很方便地生成火焰图，并且可以在浏览器里直接查看。

* profiler命令wiki: https://alibaba.github.io/arthas/profiler.html

`profiler` 命令基本运行结构是 `profiler action [actionArg]`。 下面介绍如何使用。

### 启动profiler

```
$ profiler start
Started [cpu] profiling
```

> 默认情况下，生成的是cpu的火焰图，即event为`cpu`。可以用`--event`参数来指定。

### 获取已采集的sample的数量

```
$ profiler getSamples
23
```

### 查看profiler状态

```bash
$ profiler status
[cpu] profiling is running for 4 seconds
```

可以查看当前profiler在采样哪种`event`和采样时间。

### 生成svg格式结果

```
$ profiler stop
profiler output file: /tmp/demo/arthas-output/20191125-135546.svg
OK
```

默认情况下，生成的结果保存到应用的`工作目录`下的`arthas-output`目录里。

### 通过浏览器查看arthas-output下面的profiler结果

默认情况下，arthas使用3658端口，则可以打开： [http://localhost:3658/arthas-output/](http://localhost:3658/arthas-output/) 查看到`arthas-output`目录下面的profiler结果：

![](https://alibaba.github.io/arthas/_images/arthas-output.jpg)

点击可以查看具体的结果：

![](https://alibaba.github.io/arthas/_images/arthas-output-svg.jpg)

> 如果是chrome浏览器，可能需要多次刷新。

## grep命令支持更丰富的选项

标准的linux grep命令支持丰富的选项，可以很方便地定位结果的上下文等。

新版本的`grep`命令支持更多标准的选项，下面是一些例子:

```bash
sysprop | grep java
sysprop | grep java -n
sysenv | grep -v JAVA
sysenv | grep -e "(?i)(JAVA|sun)" -m 3  -C 2
sysenv | grep JAVA -A2 -B3
thread | grep -m 10 -e  "TIMED_WAITING|WAITING"
```

感谢社区里 @qxo 的贡献。

## telnet/http协议共用3658端口

默认情况下，Arthas的Telnet端口是3658，HTTP端口是8563，这个常常让用户迷惑。在新版本里，在3658端口同时支持Telnet/HTTP协议。

在浏览器里访问 http://localhost:3658/ 也可以访问到Web Console了。

在后续的版本里，考虑默认只侦听 3658端口，减少用户的配置项。

## monitor/tt/trace等命令提供更精确的时间统计

以前Arthas被诟病比较多的一个问题是，monitor/tt/trace等命令时间统计误差大。因为以前只使用了一个int来保存时间，所以不精确。

在新版本里，改用一个高效的stack来保存数据，时间的准确度大大提升，欢迎大家反馈效果。

感谢社区里 @huangjIT 的贡献。

### 总结

总之，`3.1.5`版本的Arthas引入了开箱即用的Profiler/火焰图功能，欢迎大家使用反馈。

* Release Note: https://github.com/alibaba/arthas/releases/tag/arthas-all-3.1.5
* 火焰图的一个参考文章：https://openresty.org/posts/dynamic-tracing/
