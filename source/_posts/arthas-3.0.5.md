---
title: Alibaba应用诊断利器Arthas 3.0.5版本发布：提升全平台用户体验
date: 2018-11-29 20:58:28
tags:
 - java
 - arthas


categories:
  - 技术

---



Arthas从9月份开源以来，受到广大Java开发者的支持，Github Star数三个月超过6000，非常感谢用户支持。同时用户给Arthas提出了很多建议，其中反映最多的是：

1. Windows平台用户体验不好
1. Attach的进程和最终连接的进程不一致
1. 某些环境下没有安装Telnet，不能连接到Arthas Server
1. 本地启动，不需要下载远程（很多公司安全考虑）
1. 下载速度慢（默认从maven central repository下载）


在Arthas 3.0.5版本里，我们在用户体验方面做了很多改进，下面逐一介绍。

* Github: https://github.com/alibaba/arthas
* 文档：https://alibaba.github.io/arthas/

### 全平台通用的arthas-boot

`arthas-boot`是新增加的支持全平台的启动器，Windows/Mac/Linux下使用体验一致。下载后，直接`java -jar`命令启动：

```bash
wget https://alibaba.github.io/arthas/arthas-boot.jar
java -jar arthas-boot.jar
```

`arthas-boot`的功能比以前的`as.sh`更强大。

* 比如下载速度比较慢，可以指定阿里云的镜像。

    ```
    java -jar arthas-boot.jar --repo-mirror aliyun --use-http
    ```

* 比如可以通过`session-timeout`指定超时时间：

    ```
    java -jar arthas-boot.jar --session-timeout 3600
    ```
* 更多的功能可以通过`java -jar arthas-boot.jar -h`查看

`arthas-boot`在attach成功之后，会启动一个java telent client去连接Arthas Server，用户没有安装telnet的情况下也可以正常使用。


### 优先使用当前目录的Arthas

在新版本里，默认会从`arthas-boot.jar`和`as.sh`所在的目录下查找arthas home，这样子用户全量安装之后，不需要再从远程下载Arthas。

* 用户可以更方便地集成到自己的基础镜像，或者docker镜像里
* 对安全要求严格的公司，不用再担心从远程下载的问题

### Attach之前先检测端口

在之前的版本里，用户困扰最多的是，明明选择了进程A，但是实际连接到的却是进程B。

原因是之前attach了进程B，没有执行`shutdown`，下次再执行时，还是连接到进程B。

在新版本里，做了改进：

* 在attach之前，检测使用3658端口的进程
* 在Attach时，如果端口和进程不匹配，会打印出ERROR信息

```bash
$ java -jar arthas-boot.jar
[INFO] Process 1680 already using port 3658
[INFO] Process 1680 already using port 8563
* [1]: 1680 Demo
  [2]: 35542
  [3]: 82334 Demo
3
[ERROR] Target process 82334 is not the process using port 3658, you will connect to an unexpected process.
[ERROR] If you still want to attach target process 82334, Try to set a different telnet port by using --telnet-port argument.
[ERROR] Or try to shutdown the process 1680 using the telnet port first.
```

### 更好的历史命令匹配功能

* 新版本对键盘`Up/Down`有了更好的匹配机制，试用有惊喜:)

    比如执行了多次trace，但是在命令行输入 trace后，想不起来之前trace的具体类名，那么按`Up`，可以很轻松地匹配到之前的历史命令，不需要辛苦翻页。

* 新版本增加了`history`命令

### 改进Web Console的体验

* 改进对Windows的字体支持

    之前Windows下面使用的是非等宽字体，看起来很难受。新版本里统一为等宽字体。
* 增大字体，不再伤害眼睛

![image](https://user-images.githubusercontent.com/1683936/49217204-f2c8e280-f407-11e8-9314-65f04a9e55da.png)


### 新增sysenv命令

sysenv命令和sysprop类似，可以打印JVM的环境变量。

* https://alibaba.github.io/arthas/sysenv.html

### 新增ognl命令

ognl命令提供了单独执行ognl脚本的功能。可以很方便调用各种代码。

比如执行多行表达式，赋值给临时变量，返回一个List：

```java
$ ognl '#value1=@System@getProperty("java.home"), #value2=@System@getProperty("java.runtime.name"), {#value1, #value2}'
@ArrayList[
    @String[/opt/java/8.0.181-zulu/jre],
    @String[OpenJDK Runtime Environment],
]
```

* https://alibaba.github.io/arthas/ognl.html

### watch命令打印耗时，更方便定位性能瓶颈

之前watch命令只支持打印入参返回值等，新版本同时打印出调用耗时，可以很方便定位性能瓶颈。

```bash
$ watch demo.MathGame primeFactors 'params[0]'
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 22 ms.
ts=2018-11-29 17:53:54; [cost=0.131383ms] result=@Integer[-387929024]
ts=2018-11-29 17:53:55; [cost=0.132368ms] result=@Integer[-1318275764]
ts=2018-11-29 17:53:56; [cost=0.496598ms] result=@Integer[76446257]
ts=2018-11-29 17:53:57; [cost=4.9617ms] result=@Integer[1853966253]
```

* https://alibaba.github.io/arthas/watch.html

### 改进类搜索匹配功能，更好支持lambda和内部类

之前的版本里，在搜索lambda类时，或者反编绎lambda类有可能会失败。新版本做了修复。比如

```java
$ jad Test$$Lambda$1/1406718218

ClassLoader:
+-sun.misc.Launcher$AppClassLoader@5c647e05
  +-sun.misc.Launcher$ExtClassLoader@3c1491ce

Location:
/tmp/classes

/*
 * Decompiled with CFR 0_132.
 *
 * Could not load the following classes:
 *  Test
 *  Test$$Lambda$1
 */
import java.lang.invoke.LambdaForm;
import java.util.function.Consumer;

final class Test$$Lambda$1
implements Consumer {
    private Test$$Lambda$1() {
    }

    @LambdaForm.Hidden
    public void accept(Object object) {
        Test.lambda$0((Integer)((Integer)object));
    }
}
```

### 更好的tab自动补全

改进了很多命令的`tab`自动补全功能，有停顿时，可以多按`tab`尝试下。

### Release Note

详细的Release Note：https://github.com/alibaba/arthas/releases/tag/arthas-all-3.0.5
