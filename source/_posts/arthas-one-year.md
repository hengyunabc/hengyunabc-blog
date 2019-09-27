---
title: Arthas开源一周年，Github Star 16K，我们一直在坚持什么？
date: 2019-09-27 20:58:28
tags:
 - arthas
 - java
 - logger


categories:
  - 技术

---



## 缘起

最近看到一个很流行的标题，《开源XX年，star XXX，我是如何坚持的》。

看到这样的标题，忽然发觉Arthas从2018年9月开源以来，刚好一年了，正好在这个秋高气爽的时节做下总结和回顾。

![Arthas](https://alibaba.github.io/arthas/_images/arthas.png)

`Arthas`是Alibaba开源的Java诊断工具，深受开发者喜爱。

* Github： [https://github.com/alibaba/arthas](https://github.com/alibaba/arthas)
* 文档：[https://alibaba.github.io/arthas](https://alibaba.github.io/arthas)


回顾Arthas Star数的历史，一直保持快速增长，目前已经突破16K。

![Arthas Github Star历史曲线](https://user-images.githubusercontent.com/1683936/65661356-05ca3480-e064-11e9-92ea-8cc21e45a71f.png)

感谢用户的支持，既是压力也是动力。在过去开源的一年里，Arthas发布了7个Release版本，我们一直坚持三点：

* 持续改进易用性
* 持续增加好用的命令
* 从开源社区中获取力量，回报社区

## 持续改进易用性

Arthas一直把易用性放在第一位，在开源之后，我们做了下面的改进：

* 开发arthas boot，支持Windows/Linux/Mac统一体验
* 丝滑的自动补全，参考了jshell的体验
* 高效的历史命令匹配，`Up/Down`直达
* 改进类搜索匹配功能，更好支持lambda和内部类
* 完善重定向机制
* 支持JDK 9/10/11
* 支持Docker
* 支持rpm/deb包安装


尽管我们在易用性下了很大的功夫，但是发现很多时候用户比较难入门，因此，我们参考了k8s的 Interactive Tutorial，推出了Arthas的在线教程：

* [Arthas基础教程](https://alibaba.github.io/arthas/arthas-tutorials?language=cn&id=arthas-basics)
* [Arthas进阶教程](https://alibaba.github.io/arthas/arthas-tutorials?language=cn&id=arthas-advanced)

通过基础教程，可以在交互终端里一步步入门，通过进阶教程可以深入理解Arthas排查问题的案例。

另外，为了方便用户大规模部署，我们实现了tunnel server和用户数据回报功能：

* 增加tunnel server，统一管理Agent连接
* 增加用户数据回报功能，方便做安全管控



## 持续增加好用的命令

Arthas号称是Java应用诊断利器，那么我们自己要对得起这个口号。在开源之后，Arthas持续增加了10多个命令。

* ognl      命令任意代码执行
* mc        线上内存编译器
* redefine  命令线上热更新代码
* logger    命令一键查看应用里的所有logger配置
* sysprop   查看更新System Properties
* sysenv    查看环境变量
* vmoption  查看更新VM option
* logger    查看logger配置，更新level
* mbean     查看JMX信息
* heapdump  堆内存快照

下面重点介绍两个功能。

### jad/mc/redefine 一条龙热更新线上代码

以 [Arthas在线教程](https://alibaba.github.io/arthas/arthas-tutorials?language=cn&id=arthas-advanced) 里的`UserController`为例：

1. 使用jad反编译代码

    ```bash
    jad --source-only com.example.demo.arthas.user.UserController > /tmp/UserController.java
    ```

2. 使用vim编译代码

    当 user id 小于1时，也正常返回，不抛出异常：

    ```java
        @GetMapping("/user/{id}")
        public User findUserById(@PathVariable Integer id) {
            logger.info("id: {}" , id);

            if (id != null && id < 1) {
                return new User(id, "name" + id);
                // throw new IllegalArgumentException("id < 1");
            } else {
                return new User(id, "name" + id);
            }
        }
    ```
3. 使用`mc`命令编译修改后的`UserController.java`

    ```bash
    $ mc /tmp/UserController.java -d /tmp
    Memory compiler output:
    /tmp/com/example/demo/arthas/user/UserController.class
    Affect(row-cnt:1) cost in 346 ms
    ```
4. 使用`redefine`命令，因为可以热更新代码

    ```
    $ redefine /tmp/com/example/demo/arthas/user/UserController.class
    redefine success, size: 1
    ```

### 通过logger命令查看配置，修改level

在网站压力大的时候（比如双11），有个缓解措施就是把应用的日志level修改为ERROR。 那么有两个问题：

* 复杂应用的日志系统可能会有多个，那么哪个日志系统配置真正生效了？
* 怎样在线上动态修改logger的level？

通过`logger`命令，可以查看应用里logger的详细配置信息，比如`FileAppender`输出的文件，`AsyncAppender`是否`blocking`。

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
                                        blocking        false
                                        appenderRef     [APPLICATION]
```

也可以在线修改logger的level：

```bash
[arthas@2062]$ logger --name ROOT --level debug
update logger level success.
```


## 从开源社区中获取力量，回报社区

### 感谢67位Contributors

Arthas开源以来，一共有67位 Contributors，感谢他们贡献的改进：

 ![Arthas Contributors](https://opencollective.com/arthas/contributors.svg?width=890&button=false)

社区提交了一系列的改进，下面列出一些点（不完整）：

* 翻译了大部分英文文档的
* trace命令支持行号
* 打包格式支持rpm/deb
* 改进命令行提示符为 `arthas@pid`
* 改进对windows的支持
* 增加`mbean`命令
* 改进webconsole的体验

另外，有83个公司/组织登记了他们的使用信息，欢迎更多的用户来登记：

![Arthas Users](https://user-images.githubusercontent.com/1683936/65661477-5772bf00-e064-11e9-91c1-5cd32b09e7c6.png)


### 洐生项目

基于Arthas，还产生了一些洐生项目，下面是其中两个：

* Bistoury: 去哪儿网开源的集成了Arthas的项目
* arthas-mvel: 一个使用MVEL脚本的fork

### 用户案例分享

广大用户在使用Arthas排查问题过程中，分享了很多排查过程和心得，欢迎大家来分享。

![Arthas用户案例分享](https://user-images.githubusercontent.com/1683936/65661495-5cd00980-e064-11e9-98e5-4b76956af1c4.png)

### 回馈开源

Arthas本身使用了很多开源项目的代码，在开源过程中，我们给netty, ognl, cfr等都贡献了改进代码，回馈上游。

## 后记

 在做Arthas宣传小册子时，Arthas的宣传语是：
 
 > “赠人玫瑰之手，经久犹有余香”
 
希望Arthas未来能帮助到更多的用户解决问题，也希望广大的开发者对Arthas提出更多的改进和建议。

最后是抽奖环节，大家可以转发文章，在公众号后台留言自己和Arthas的故事，或者给Arthas提出建议，奖品是Arthas的卫衣一件：

![Arthas卫衣](https://user-images.githubusercontent.com/1683936/65737523-619ec700-e111-11e9-8e3f-f91edfd33084.png)


## 公众号

欢迎关注横云断岭的专栏，专注Java，Spring Boot，Arthas，Dubbo。

![横云断岭的专栏](http://hengyunabc.github.io/img/qrcode_gongzhonghao.jpg)


