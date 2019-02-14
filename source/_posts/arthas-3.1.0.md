---
title: 深入Spring Boot系列
date: 2019-02-13 16:58:28
tags:
 - arthas
 - java
 - bytecode


categories:
  - 技术

---


![Arthas](https://alibaba.github.io/arthas/_images/arthas.png)

从Arthas上个版本发布，已经过去两个多月了，Arthas 3.1.0版本不仅带来大家投票出来的新LOGO，还带来强大的新功能和更好的易用性，下面一一介绍。

### 在线教程

在新版本Arthas里，增加了在线教程，用户可以在线运行Demo，一步步学习Arthas的各种用法，推荐新手尝试：

* [Arthas基础教程](https://alibaba.github.io/arthas/arthas-tutorials?language=cn&id=arthas-basics)
* [Arthas进阶教程](https://alibaba.github.io/arthas/arthas-tutorials?language=cn&id=arthas-advanced)

非常欢迎大家来完善这些教程。

### 增加内存编绎器支持，在线编辑热更新代码

`3.1.0`版本里新增命令`mc`，不是方块游戏mc，而是Memory Compiler。

在之前版本里，增加了`redefine`命令，可以热更新字节码。但是有个不方便的地方：需要把`.class`文件上传到服务器上。

在`3.1.0`版本里，结合`jad`/`mc`/`redefine` 可以完美实现热更新代码。

以 [Arthas在线教程](https://alibaba.github.io/arthas/arthas-tutorials?language=cn&id=arthas-advanced) 里的`UserController`为例：

1. 使用jad反编绎代码

    ```bash
    jad --source-only com.example.demo.arthas.user.UserController > /tmp/UserController.java
    ```

2. 使用vim编绎代码

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
3. 使用`mc`命令编绎修改后的`UserController.java`

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

### 丝滑的自动补全

在新版本里，改进了很多命令的自动补全，比如 `watch/trace/tt/monitor/stack`等。

下面是watch命令的第一个`Tab`补全结果，用户可以很方便的一步步补全类名，函数名：

```bash
$ watch
com.   sun.   javax. ch.    io.    demo.  jdk.   org.   java.
```

另外，新增加了 `jad/sc/sm/redefine` 等命令的自动补全支持，多按`Tab`有惊喜。

### 新版本的Web console

新版本的Web Console切换到了`xtermd.js`，更好地支持现代浏览器。

* 支持`Ctrl + C`复制
* 支持全屏

![web console](https://alibaba.github.io/arthas/_images/web-console-local.png)

### Docker镜像支持

Arthas支持Docker镜像了

* 用户可以很方便地诊断Docker/k8s里的Java进程
* 也可以很方便地把Arthas加到自己的基础镜像里

参考： https://alibaba.github.io/arthas/docker.html

### 重定向重新设计

之前的版本里，Arthas的重定向是会放到一个`~/logs/arthas-cache/`目录里，违反直觉。

在新版本里，重定向和Linux下面的一致，`>`/`>>`的行为也和Linux下一致。

并且，增加了 `cat`/`pwd`命令，可以配置使用。


### 总结

总之，`3.1.0`版本的Arthas带了非常多的新功能，改进了很多的用户体验，欢迎大家使用反馈。

* Arthas在线教程可以学到很多技巧
* jad/mc/redefine 一条龙非常强大
* 丝滑的自动补全值得尝试
* 新版本的Web console有惊奇

Release Note: https://github.com/alibaba/arthas/releases/tag/3.1.0