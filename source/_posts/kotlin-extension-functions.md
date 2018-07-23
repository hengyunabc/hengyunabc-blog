---
title: Kotlin里的Extension Functions实现原理分析
date: 2018-07-24 00:58:28
tags:
 - kotlin
 - bytecode
 - asm
 - java


categories:
  - 技术

---


## Kotlin里的Extension Functions



Kotlin里有所谓的扩展函数(Extension Functions)，支持给现有的java类增加函数。

* https://kotlinlang.org/docs/reference/extensions.html

比如给`String`增加一个`hello`函数，可以这样子写：

```java
fun String.hello(world : String) : String {
    return "hello " + world + this.length;
}
fun main(args: Array<String>) {
    System.out.println("abc".hello("world"));
}
```

可以看到在main函数里，直接可以在`String`上调用`hello`函数。

执行后，输出结果是：

```
hello world3
```

可以看到在`hello`函数里的`this`引用的是`"abc"`。


刚开始看到这个语法还是比较新奇的，那么怎样实现的呢？**如果不同的库都增加了同样的函数会不会冲突？**

反编绎生成的字节码，结果是：

```java
  @NotNull
  public static final String hello(@NotNull String $receiver, @NotNull String world) {
    return "hello " + world + $receiver.length();
  }
  public static final void main(@NotNull String[] args) {
    System.out.println(hello("abc", "world"));
  }
```

可以看到，实际上是增加了一个static public final函数。

**并且新增加的函数是在自己的类里的，并不是在String类里。即不同的库新增加的扩展函数都是自己类里的，不会冲突。**


## lombok 里的 @ExtensionMethod 实现

lombok里也提供了类似的`@ExtensionMethod`支持。

* https://projectlombok.org/features/experimental/ExtensionMethod


和上面的例子一样，给String类增加一个`hello`函数：

* 需要定义一个`class Extensions`
* 再用`@ExtensionMethod`声明

```java
class Extensions {
    public static String hello(String receiver, String world) {
        return "hello " + world + receiver.length();
    }
}
@ExtensionMethod({
        Extensions.class
})

public class Test {
    public static void main(String[] args) {
        System.out.println("abc".hello("world"));
    }
}
```

执行后，输出结果是：

```
hello world3
```

可以看到在`hello`函数里，第一个参数`String receiver`就是`"abc"`本身。

和上面kotlin的例子不一样的是，kotlin里直接可以用`this`。


生成的字节码反编绎结果是：

```java
class Extensions {
  public static String hello(String receiver, String world) {
    return "hello " + world + receiver.length();
  }
}
public class Test {
  public static void main(String[] args) {
    System.out.println(Extensions.hello("abc", "world"));
  }
}
```

可以看到所谓的`@ExtensionMethod`实际上也是一个语法糖。

## 设计动机

* https://kotlinlang.org/docs/reference/extensions.html#motivation 

据kotlin的文档：各种`FileUtils`，`StringUtils`类很麻烦。

比如像下面处理`List`，在java里可以用`java.util.Collections`

```java
// Java
Collections.swap(list, Collections.binarySearch(list,
    Collections.max(otherList)),
    Collections.max(list));
```

简化下import，可以变为

```java
// Java
swap(list, binarySearch(list, max(otherList)), max(list));
```

但是还不够清晰，各种`import *`也是比较烦的。利用扩展函数，可以直接这样子写：

```java
// Java
list.swap(list.binarySearch(otherList.max()), list.max());
```

## 总结

* kotlin的Extension Functions和lombok的`@ExtensionMethod`实际上都是增加public static final函数
* 不同的库增加的同样的Extension Functions不会冲突
* 设计的动机是减少各种utils类。