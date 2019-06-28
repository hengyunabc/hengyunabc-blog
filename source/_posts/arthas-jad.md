---
title: Arthas源码分析--jad反编译原理
date: 2019-06-27 20:58:28
tags:
 - arthas
 - jad
 - java
 - bytecode


categories:
  - 技术

---


Arthas是阿里巴巴开源的Java应用诊断利器，本文介绍Arthas 3.1.1版本里`jad`命令的实现原理。

* https://github.com/alibaba/arthas

## `jad`命令介绍

* https://alibaba.github.io/arthas/jad.html

jad即java decompiler，把JVM已加载类的字节码反编译成Java代码。比如反编译`String`类：

```java
$ jad java.lang.String
 
ClassLoader:
 
Location:
 
/*
* Decompiled with CFR .
*/
package java.lang;
 
import java.io.ObjectStreamField;
...
public final class String
implements Serializable,
Comparable<String>,
CharSequence {
    private final char[] value;
    private int hash;
    private static final long serialVersionUID = -6849794470754667710L;
    private static final ObjectStreamField[] serialPersistentFields = new ObjectStreamField[0];
    public static final Comparator<String> CASE_INSENSITIVE_ORDER = new CaseInsensitiveComparator();
 
    public String(byte[] arrby, int n, int n2) {
        String.checkBounds(arrby, n, n2);
        this.value = StringCoding.decode(arrby, n, n2);
    }
...
```

## 获取到类的字节码

反编译有两部分工作：

1. 获取到字节码
2. 反编译为Java代码

那么怎么从运行的JVM里获取到字节码？

最常见的思路是，在`classpaths`下面查找，比如 `ClassLoader.getResource("java/lang/String.class")`，但是这样子查找到的字节码不一定对。比如可能有多个冲突的jar，或者有Java Agent修改了字节码。


## ClassFileTransformer机制

从JDK 1.5起，有一套`ClassFileTransformer`的机制，Java Agent通过`Instrumentation`注册`ClassFileTransformer`，那么在类加载或者`retransform`时就可以回调修改字节码。

显然，在Arthas里，要增强的类是已经被加载的，所以它们的字节码都是在`retransform`时被修改的。
通过显式调用`Instrumentation.retransformClasses(Class<?>...)`可以触发回调。

Arthas里增强字节码的`watch`/`trace`/`stack`/`tt`等命令都是通过`ClassFileTransformer`来实现的。

`ClassFileTransformer`的接口如下：

```java
public interface ClassFileTransformer {
    byte[]
    transform(  ClassLoader         loader,
                String              className,
                Class<?>            classBeingRedefined,
                ProtectionDomain    protectionDomain,
                byte[]              classfileBuffer)
        throws IllegalClassFormatException;
```

看到这里，读者应该猜到`jad`是怎么获取到字节码的了：

1. 注册一个`ClassFileTransformer`
2. 通过`Instrumentation.retransformClasses`触发回调
3. 在回调的`transform`函数里获取到字节码
4. 删掉注册的`ClassFileTransformer`

## 使用cfr来反编译

获取到字节码之后，怎样转换为Java代码呢？

以前大家使用比较多的反编译软件可能是`jd-gui`，但是它不支持JDK8的lambda语法和一些新版本JDK的特性。

后面比较成熟的反编译软件是`cfr`，它以前是不开源的。直到最近的`0.145`版本，作者终于开源了，可喜可贺。地址是

* https://github.com/leibnitz27/cfr

在Arthas `jad`命令里，通过调用`cfr`来完成反编译。

## `jad`命令的缺陷

99%的情况下，`jad`命令dump下来的字节码是准确的，除了一些极端情况。

1. 因为JVM里注册的`ClassFileTransformer`可能有多个，那么在JVM里运行的字节码里，可能是被多个`ClassFileTransformer`处理过的。
2. 触发了`retransformClasses`之后，这些注册的`ClassFileTransformer`会被依次回，上一个处理的字节码传递到下一个。
所以不能保证这些`ClassFileTransformer`第二次执行会返回同样的结果。
3. 有可能一些`ClassFileTransformer`会被删掉，触发`retransformClasses`之后，之前的一些修改就会丢失掉。

所以目前在Arthas里，如果开两个窗口，一个窗口执行`watch`/`tt`等命令，另一个窗口对这个类执行`jad`，那么可以观察到`watch`/`tt`停止了输出，实际上是因为字节码在触发了`retransformClasses`之后，`watch`/`tt`所做的修改丢失了。

## 精确获取JVM内运行的java字节码的办法

如果想精确获取到JVM内运行的Java字节码，可以使用这个`dumpclass`工具，它是通过`sa-jdi.jar`来实现的，保证dump下来的字节码是JVM内所运行的。

* https://github.com/hengyunabc/dumpclass


## 总结

总结`jad`命令的工作原理：

* 通过注册`ClassFileTransformer`，再触发`retransformClasses`来获取字节码
* 通过`cfr`来反编译
* `ClassFileTransformer`的方式来获取字节码有一定缺陷
* 通过`dumpclass`工具可以精确获取字节码

`jad`命令可以在线上快速检查运行时的代码，并且结合`mc`/`redefine`命令可以热更新代码：

* jad/mc/redefine线上热更新一条龙: http://hengyunabc.github.io/arthas-online-hotswap/

## 链接
* Arthas开源：https://github.com/alibaba/arthas
* dumpclass: https://github.com/hengyunabc/dumpclass
* https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/ClassFileTransformer.html
