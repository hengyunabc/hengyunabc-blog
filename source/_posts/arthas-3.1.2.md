---
title: Alibaba Arthas 3.1.2版本：增加logger/heapdump/vmoption命令，支持tunnel server
date: 2019-09-09 20:58:28
tags:
 - arthas
 - java
 - logger


categories:
  - 技术

---


![Arthas](https://alibaba.github.io/arthas/_images/arthas.png)

`Arthas`是Alibaba开源的Java诊断工具，深受开发者喜爱。

* Github： https://github.com/alibaba/arthas
* 文档：https://alibaba.github.io/arthas

Arthas 3.1.2版本持续增加新特性，下面重点介绍：

* `logger/heapdump/vmoption/stop`命令
* 通过tunnel server连接不同网络的arthas，方便统一管控
* 易用性持续提升：提示符修改为`arthas@pid`形式，支持`ctrl + k`清屏快捷键



## logger/heapdump/vmoption/stop命令

### logger命令

> 查看logger信息，更新logger level

* https://alibaba.github.io/arthas/logger.html

#### 查看所有logger信息

以下面的`logback.xml`为例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="APPLICATION" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>mylog-%d{yyyy-MM-dd}.%i.txt</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>60</maxHistory>
            <totalSizeCap>2GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%logger{35} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="APPLICATION" />
    </appender>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%-4relative [%thread] %-5level %logger{35} - %msg %n
            </pattern>
            <charset>utf8</charset>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="ASYNC" />
    </root>
</configuration>
```


使用`logger`命令打印的结果是：

```bash
[arthas@2062]$ logger
 name                                   ROOT
 class                                  ch.qos.logback.classic.Logger
 classLoader                            sun.misc.Launcher$AppClassLoader@2a139a55
 classLoaderHash                        2a139a55
 level                                  INFO
 effectiveLevel                         INFO
 additivity                             true
 codeSource                             file:/Users/hengyunabc/.m2/repository/ch/qos/logback/logback-classic/1.2.3/logback-classic-1.2.3.jar
 appenders                              name            CONSOLE
                                        class           ch.qos.logback.core.ConsoleAppender
                                        classLoader     sun.misc.Launcher$AppClassLoader@2a139a55
                                        classLoaderHash 2a139a55
                                        target          System.out
                                        name            APPLICATION
                                        class           ch.qos.logback.core.rolling.RollingFileAppender
                                        classLoader     sun.misc.Launcher$AppClassLoader@2a139a55
                                        classLoaderHash 2a139a55
                                        file            app.log
                                        name            ASYNC
                                        class           ch.qos.logback.classic.AsyncAppender
                                        classLoader     sun.misc.Launcher$AppClassLoader@2a139a55
                                        classLoaderHash 2a139a55
                                        appenderRef     [APPLICATION]
```

从`appenders`的信息里，可以看到

* `CONSOLE` logger的target是`System.out`
* `APPLICATION` logger是`RollingFileAppender`，它的file是`app.log`
* `ASYNC`它的`appenderRef`是`APPLICATION`，即异步输出到文件里


#### 查看指定名字的logger信息

```bash
[arthas@2062]$ logger -n org.springframework.web
 name                                   org.springframework.web
 class                                  ch.qos.logback.classic.Logger
 classLoader                            sun.misc.Launcher$AppClassLoader@2a139a55
 classLoaderHash                        2a139a55
 level                                  null
 effectiveLevel                         INFO
 additivity                             true
 codeSource                             file:/Users/hengyunabc/.m2/repository/ch/qos/logback/logback-classic/1.2.3/logback-classic-1.2.3.jar
```

#### 更新logger level

```bash
[arthas@2062]$ logger --name ROOT --level debug
update logger level success.
```

### heapdump命令

> dump java heap, 类似jmap命令的heap dump功能。

* https://alibaba.github.io/arthas/heapdump.html

#### dump到指定文件

```bash
[arthas@58205]$ heapdump /tmp/dump.hprof
Dumping heap to /tmp/dump.hprof...
Heap dump file created
```

#### 只dump live对象

```bash
[arthas@58205]$ heapdump --live /tmp/dump.hprof
Dumping heap to /tmp/dump.hprof...
Heap dump file created
```


### vmoption命令

> 查看，更新VM诊断相关的参数

* https://alibaba.github.io/arthas/vmoption.html

#### 查看所有的option

```bash
[arthas@56963]$ vmoption
 KEY                    VALUE                   ORIGIN                 WRITEABLE
---------------------------------------------------------------------------------------------
 HeapDumpBeforeFullGC   false                   DEFAULT                true
 HeapDumpAfterFullGC    false                   DEFAULT                true
 HeapDumpOnOutOfMemory  false                   DEFAULT                true
 Error
 HeapDumpPath                                   DEFAULT                true
 CMSAbortablePrecleanW  100                     DEFAULT                true
 aitMillis
 CMSWaitDuration        2000                    DEFAULT                true
 CMSTriggerInterval     -1                      DEFAULT                true
 PrintGC                false                   DEFAULT                true
 PrintGCDetails         true                    MANAGEMENT             true
 PrintGCDateStamps      false                   DEFAULT                true
 PrintGCTimeStamps      false                   DEFAULT                true
 PrintGCID              false                   DEFAULT                true
 PrintClassHistogramBe  false                   DEFAULT                true
 foreFullGC
 PrintClassHistogramAf  false                   DEFAULT                true
 terFullGC
 PrintClassHistogram    false                   DEFAULT                true
 MinHeapFreeRatio       0                       DEFAULT                true
 MaxHeapFreeRatio       100                     DEFAULT                true
 PrintConcurrentLocks   false                   DEFAULT                true
```

#### 查看指定的option

```bash
[arthas@56963]$ vmoption PrintGCDetails
 KEY                    VALUE                   ORIGIN                 WRITEABLE
---------------------------------------------------------------------------------------------
 PrintGCDetails         false                   MANAGEMENT             true
```

#### 更新指定的option

```bash
[arthas@56963]$ vmoption PrintGCDetails true
Successfully updated the vm option.
PrintGCDetails=true
```

### stop命令

之前有用户吐槽，不小心退出Arthas console之后，`shutdown`会关闭系统，因些增加了`stop`命令来退出arthas，功能和`shutdown`命令一致。


## 通过tunnel server连接不同网络的arthas

* https://alibaba.github.io/arthas/web-console.html

在新版本里，增加了arthas tunnel server的功能，用户可以通过tunnel server很方便连接不同网络里的arthas agent，适合做统一管控。

### 启动arthas时连接到tunnel server

在启动arthas，可以传递`--tunnel-server`参数，比如：

```bash
as.sh --tunnel-server 'ws://47.75.156.201:7777/ws'
```

> 目前`47.75.156.201`是一个测试服务器，用户可以自己搭建arthas tunnel server

* 如果有特殊需求，可以通过`--agent-id`参数里指定agentId。默认情况下，会生成随机ID。

attach成功之后，会打印出agentId，比如：

```bash
  ,---.  ,------. ,--------.,--.  ,--.  ,---.   ,---.
 /  O  \ |  .--. ''--.  .--'|  '--'  | /  O  \ '   .-'
|  .-.  ||  '--'.'   |  |   |  .--.  ||  .-.  |`.  `-.
|  | |  ||  |\  \    |  |   |  |  |  ||  | |  |.-'    |
`--' `--'`--' '--'   `--'   `--'  `--'`--' `--'`-----'


wiki      https://alibaba.github.io/arthas
tutorials https://alibaba.github.io/arthas/arthas-tutorials
version   3.1.2
pid       86183
time      2019-08-30 15:40:53
id        URJZ5L48RPBR2ALI5K4V
```

如果是启动时没有连接到 tunnel server，也可以在后续自动重连成功之后，通过 session命令来获取 agentId：

```bash
[arthas@86183]$ session
 Name           Value
-----------------------------------------------------
 JAVA_PID       86183
 SESSION_ID     f7273eb5-e7b0-4a00-bc5b-3fe55d741882
 AGENT_ID       URJZ5L48RPBR2ALI5K4V
 TUNNEL_SERVER  ws://47.75.156.201:7777/ws
```


以上面的为例，在浏览器里访问 [http://47.75.156.201:8080/](http://47.75.156.201:8080/) ，输入 `agentId`，就可以连接到本机上的arthas了。


![](https://alibaba.github.io/arthas/_images/arthas-tunnel-server.png)


#### Arthas tunnel server的工作原理

```
browser <-> arthas tunnel server <-> arthas tunnel client <-> arthas agent
```

[https://github.com/alibaba/arthas/blob/master/tunnel-server/README.md](https://github.com/alibaba/arthas/blob/master/tunnel-server/README.md)


## 易用性持续提升

* 提示符修改为`arthas@pid`形式，用户可以确定当前进程ID，避免多个进程时误操作

    ```
    [arthas@86183]$ help
    ```

* 增加`ctrl + k`清屏快捷键

### 总结

总之，`3.1.2`版本的Arthas新增加了`logger/heapdump/vmoption/stop`命令，增加了tunnel server，方便统一管控。另外还有一些bug修复等，可以参考

* Release Note: https://github.com/alibaba/arthas/releases/tag/3.1.2

最后，Arthas的在线教程考虑重新组织，欢迎大家参与，提出建议：

* https://github.com/alibaba/arthas/issues/847