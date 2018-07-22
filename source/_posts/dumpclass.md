---
title: 从java进程里dump出类的class文件的小工具--dumpclass
date: 2016-04-09 21:58:28
tags:
 - java
 - jvm


categories:
  - 技术

---


## Serviceability Agent

想要查看一些被增强过的类的字节码，或者一些AOP框架的生成类，就需要dump出运行时的java进程里的字节码。

从运行的java进程里dump出运行中的类的class文件的方法，所知道的有两种

* 用agent attatch 到进程，然后利用`Instrumentation`和`ClassFileTransformer`就可以获取 到类的字节码了。

* 使用`sd-jdi.jar`里的工具

`sd-jdi.jar` 里自带的的`sun.jvm.hotspot.tools.jcore.ClassDump`就可以把类的class内容dump到文件里。

`ClassDump`里可以设置两个System properties：

* `sun.jvm.hotspot.tools.jcore.filter`  Filter的类名
* `sun.jvm.hotspot.tools.jcore.outputDir` 输出的目录

`sd-jdi.jar` 里有一个`sun.jvm.hotspot.tools.jcore.PackageNameFilter`，可以指定Dump哪些包里的类。`PackageNameFilter`里有一个System property可以指定过滤哪些包：`sun.jvm.hotspot.tools.jcore.PackageNameFilter.pkgList`。

所以可以通过这样子的命令来使用：

```
sudo java -classpath "$JAVA_HOME/lib/sa-jdi.jar" -Dsun.jvm.hotspot.tools.jcore.filter=sun.jvm.hotspot.tools.jcore.PackageNameFilter -Dsun.jvm.hotspot.tools.jcore.PackageNameFilter.pkgList=com.test  sun.jvm.hotspot.tools.jcore.ClassDump
```

显然，这个使用起来太麻烦了，而且不能应对复杂的场景。

## dumpclass

dumpclass这个小工具做了一些增强，更加方便地使用。

* 支持`? *`的匹配
* 支持多个ClassLoader加载了同名类的情况。

比如多个classloader加载了多份的logger，如果不做区分，则dump出来时会被覆盖掉，也分析不出问题。

dumpclass可以在maven仓库里下载到：
http://search.maven.org/#search%7Cga%7C1%7Cdumpclass

dumpclass的用法很简单，比如：

```bash
Usage:
 java -jar dumpclass.jar <pid> <pattern> [outputDir] <--classLoaderPrefix>

Example:
 java -jar dumpclass.jar 4345 *StringUtils
 java -jar dumpclass.jar 4345 *StringUtils /tmp
 java -jar dumpclass.jar 4345 *StringUtils /tmp --classLoaderPrefix
```

对于多个ClassLoader的情况，可以使用`--classLoaderPrefix`，这样子在输出`.class`文件时，会为每一个ClasssLoader创建一个目录，比如：`sun.jvm.hotspot.oops.Instance@955d26b8`。并且会在目录下放一个`classLoader.text`文件，里面是`ClassLoader.toString()`的内容，方便查看具体ClassLoader是什么。


源码和文档：

https://github.com/hengyunabc/dumpclass 


## HSDB

在`sa-jdi.jar`里，还有一个图形化的工具HSDB，也可以用来查看运行的的字节码。

```bash
sudo java -classpath "$JAVA_HOME/lib/sa-jdi.jar" sun.jvm.hotspot.HSDB
```


## 参考

http://rednaxelafx.iteye.com/blog/727938 
https://docs.oracle.com/javase/7/docs/api/java/lang/instrument/ClassFileTransformer.html
http://openjdk.java.net/groups/hotspot/docs/Serviceability.html 