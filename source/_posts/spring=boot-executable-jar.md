---
title: spring boot executable jar/war 原理
date: 2016-04-03 20:58:28
sticky: 10
tags:
 - spring
 - spring-boot
 - java


categories:
  - 技术

---


## spring boot executable jar/war

spring boot里其实不仅可以直接以 `java -jar demo.jar`的方式启动，还可以把jar/war变为一个可以执行的脚本来启动，比如`./demo.jar`。

把这个executable jar/war 链接到`/etc/init.d`下面，还可以变为linux下的一个service。

只要在`spring boot maven plugin`里配置：

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <executable>true</executable>
    </configuration>
</plugin>
```

这样子打包出来的jar/war就是可执行的。更多详细的内容可以参考官方的文档。

http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#deployment-install

## zip格式里的magic number

生成的jar/war实际上是一个zip格式的文件，这个zip格式文件为什么可以在shell下面直接执行？

研究了下zip文件的格式。zip文件是由entry组成的，而每一个entry开头都有一个4个字节的magic number：

```
Local file header signature = 0x04034b50 (read as a little-endian number)

即 PK\003\004
```

参考：https://en.wikipedia.org/wiki/Zip_(file_format)

zip处理软件是读取到magic number才开始处理。所以在linux/unix下面，可以把一个bash文件直接写在一个zip文件的开头，这样子会被认为是一个bash script。 而zip处理软件在读取这个文件时，仍然可以正确地处理。

比如spring boot生成的executable jar/war，的开头是：

```bash
#!/bin/bash
#
#    .   ____          _            __ _ _
#   /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
#  ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
#   \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
#    '  |____| .__|_| |_|_| |_\__, | / / / /
#   =========|_|==============|___/=/_/_/_/
#   :: Spring Boot Startup Script ::
#
```

在script内容结尾，可以看到zip entry的magic number:

```
exit 0
PK^C^D
```

## spring boot的launch.script

实际上spring boot maven plugin是把下面这个script打包到fat jar的最前面部分。

https://github.com/spring-projects/spring-boot/blob/1ca9cdabf71f3f972a9c1fdbfe9a9f5fda561287/spring-boot-tools/spring-boot-loader-tools/src/main/resources/org/springframework/boot/loader/tools/launch.script 

这个launch.script 支持很多变量设置。还可以自动识别是处于`auto`还是`service`不同mode中。

所谓的`auto mode`就是指直接运行jar/war：

```bash
./demo.jar
```

而`service mode`则是由操作系统在启动service的情况：

```bash
service demo start/stop/restart/status
```

所以fat jar可以直接在普通的命令行里执行，`./xxx.jar` 或者link到`/etc/init.d/`下，变为一个service。



