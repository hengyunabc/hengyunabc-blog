---
title: 从JVM heap dump里查找没有关闭文件的引用
date: 2018-07-02 16:01:41
tags:
 - java
 - jvm
 - fd
 - visualvm

categories:
 - 技术
---


### 背景

最近排查一个文件没有关闭的问题，记录一下。

哪些文件没有关闭是比较容易找到的，查看进程的fd(File Descriptor)就可以。但是确定fd是在哪里被打开，在哪里被引用的就复杂点，特别是在没有重启应用的情况下。
在JVM里可以通过heap dump比较方便地反查对象的引用，从而找到泄露的代码。

以下面简单的demo为例，Demo会创建一个临时文件，并且没有close掉：

```java
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
public class Test {
	public static void main(String[] args) throws IOException {
		File tempFile = File.createTempFile("test", "ttt");
		FileInputStream fi = new FileInputStream(tempFile);

		System.in.read();
	}
}
```



### 通过文件名查找对应的fd

进程打开的文件在OS里有对应的fd(File Descriptor)，可以用lsof命令或者直接在linux下到`/proc`目录下查看。

以demo为例，可以找到test文件的fd是12：

```
$ ls -alh /proc/11278/fd/
total 0
dr-x------ 2 admin users  0 Jun 30 18:20 .
dr-xr-xr-x 8 admin users  0 Jun 30 18:20 ..
lrwx------ 1 admin users 64 Jun 30 18:20 0 -> /dev/pts/0
lrwx------ 1 admin users 64 Jun 30 18:20 1 -> /dev/pts/0
lr-x------ 1 admin users 64 Jun 30 18:24 11 -> /dev/urandom
lr-x------ 1 admin users 64 Jun 30 18:24 12 -> /tmp/test7607712940880692142ttt
```

### 对进程进行heap dump

使用jmap命令：

```
jmap -dump:live,format=b,file=heap.bin 11278
```

### 通过OQL查询`java.io.FileDescriptor`对象

对于每一个打开的文件在JVM里都有一个`java.io.FileDescriptor`对象。查看下源码，可以发现`FileDescriptor`里有一个`fd`字段：

```java
public final class FileDescriptor {
    private int fd;
```

所以需要查找到fd等于12的`FileDescriptor`，QOL语句：

```
select s from java.io.FileDescriptor s where s.fd == 12
```

#### 使用VisualVM里的OQL控制台查询

在jdk8里自带VisualVM，jdk9之后可以单独下载：https://visualvm.github.io/

把heap dump文件导入VisualVM里，然后在“OQL控制台”查询上面的语句，结果是：

![visualvm-query](/img/visualvm-query.png)

再可以查询到parent，引用相关的对象。

![visualvm-object](/img/visualvm-object.png)


#### 使用jhat查询

除了VisualVM还有其它很多heap dump工具，在jdk里还自带一个jhat工具，尽管在jdk9之后移除掉了，但是个人还是比较喜欢这个工具，因为它是一个web接口的。

```
jhat -port 7000 heap.bin
```

访问 http://localhost:7000/oql/ ，可以在浏览器里查询OQL：

![jhat-query](/img/jhat-query.png)

打开链接可以查看具体的信息

![jhat-object](/img/jhat-object.png)

### 总结

* 先找出没有关闭文件的fd
* 从heap dump里据fd找出对应的`java.io.FileDescriptor`对象，再找到相关引用

### 链接

* [ViauslVM](https://visualvm.github.io/)
* [Object Query Language (OQL)](http://cr.openjdk.java.net/~sundar/8022483/webrev.01/raw_files/new/src/share/classes/com/sun/tools/hat/resources/oqlhelp.html)