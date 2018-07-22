---
title: 深入Spring Boot：ClassLoader的继承关系和影响
date: 2017-08-19 16:01:41
tags:
 - spring
 - spring-boot
 - ClassLoader
 - java

categories:
 - 技术
---

## 前言

对spring boot本身启动原理的分析，请参考：http://hengyunabc.github.io/spring-boot-application-start-analysis/

## Spring boot里的ClassLoader继承关系


可以运行下面提供的demo，分别在不同的场景下运行，可以知道不同场景下的Spring boot应用的ClassLoader继承关系。

https://github.com/hengyunabc/spring-boot-inside/tree/master/demo-classloader-context

分三种情况：

### 在IDE里，直接run main函数

则Spring的ClassLoader直接是SystemClassLoader。ClassLoader的urls包含全部的jar和自己的`target/classes`

```
========= Spring Boot Application ClassLoader Urls =============
ClassLoader urls: sun.misc.Launcher$AppClassLoader@2a139a55
file:/Users/hengyunabc/code/java/spring-boot-inside/demo-classloader-context/target/classes/
file:/Users/hengyunabc/.m2/repository/org/springframework/cloud/spring-cloud-starter/1.1.9.RELEASE/spring-cloud-starter-1.1.9.RELEASE.jar
file:/Users/hengyunabc/.m2/repository/org/springframework/boot/spring-boot-starter/1.4.7.RELEASE/spring-boot-starter-1.4.7.RELEASE.jar
...
```

### 以fat jar运行

```bash
mvn clean package
java -jar target/demo-classloader-context-0.0.1-SNAPSHOT.jar
```

执行应用的main函数的ClassLoader是`LaunchedURLClassLoader`，它的parent是`SystemClassLoader`。

```
========= ClassLoader Tree=============
org.springframework.boot.loader.LaunchedURLClassLoader@1218025c
- sun.misc.Launcher$AppClassLoader@6bc7c054
-- sun.misc.Launcher$ExtClassLoader@85ede7b
```

并且`LaunchedURLClassLoader`的urls是 fat jar里的`BOOT-INF/classes!/`目录和`BOOT-INF/lib`里的所有jar。

```
========= Spring Boot Application ClassLoader Urls =============
ClassLoader urls: org.springframework.boot.loader.LaunchedURLClassLoader@1218025c
jar:file:/Users/hengyunabc/code/java/spring-boot-inside/demo-classloader-context/target/demo-classloader-context-0.0.1-SNAPSHOT.jar!/BOOT-INF/classes!/
jar:file:/Users/hengyunabc/code/java/spring-boot-inside/demo-classloader-context/target/demo-classloader-context-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-boot-1.4.7.RELEASE.jar!/
jar:file:/Users/hengyunabc/code/java/spring-boot-inside/demo-classloader-context/target/demo-classloader-context-0.0.1-SNAPSHOT.jar!/BOOT-INF/lib/spring-web-4.3.9.RELEASE.jar!/
...
```

`SystemClassLoader`的urls是`demo-classloader-context-0.0.1-SNAPSHOT.jar`本身。

```
========= System ClassLoader Urls =============
ClassLoader urls: sun.misc.Launcher$AppClassLoader@6bc7c054
file:/Users/hengyunabc/code/java/spring-boot-inside/demo-classloader-context/target/demo-classloader-context-0.0.1-SNAPSHOT.jar
```


### 以解压目录运行

```
mvn clean package
cd target
unzip demo-classloader-context-0.0.1-SNAPSHOT.jar -d demo
cd demo
java org.springframework.boot.loader.PropertiesLauncher
```

执行应用的main函数的ClassLoader是`LaunchedURLClassLoader`，它的parent是`SystemClassLoader`。

```
========= ClassLoader Tree=============
org.springframework.boot.loader.LaunchedURLClassLoader@4aa298b7
- sun.misc.Launcher$AppClassLoader@2a139a55
-- sun.misc.Launcher$ExtClassLoader@1b6d3586
```

`LaunchedURLClassLoader`的urls是解压目录里的`BOOT-INF/classes/`和`/BOOT-INF/lib/`下面的jar包。

```
========= Spring Boot Application ClassLoader Urls =============
ClassLoader urls: org.springframework.boot.loader.LaunchedURLClassLoader@4aa298b7
file:/Users/hengyunabc/code/java/spring-boot-inside/demo-classloader-context/target/demo/BOOT-INF/classes/
jar:file:/Users/hengyunabc/code/java/spring-boot-inside/demo-classloader-context/target/demo/BOOT-INF/lib/bcpkix-jdk15on-1.55.jar!/
jar:file:/Users/hengyunabc/code/java/spring-boot-inside/demo-classloader-context/target/demo/BOOT-INF/lib/bcprov-jdk15on-1.55.jar!/
jar:file:/Users/hengyunabc/code/java/spring-boot-inside/demo-classloader-context/target/demo/BOOT-INF/lib/classmate-1.3.3.jar!/
```

`SystemClassLoader`的urls只有当前目录：

```
========= System ClassLoader Urls =============
ClassLoader urls: sun.misc.Launcher$AppClassLoader@2a139a55
file:/Users/hengyunabc/code/java/spring-boot-inside/demo-classloader-context/target/demo/
```

> 其实还有两种运行方式：`mvn spring-boot:run` 和 `mvn spring-boot:run -Dfork=true`，但是比较少使用，不单独讨论。感觉兴趣的话可以自行跑下。


### 总结spring boot里ClassLoader的继承关系

* 在IDE里main函数执行时，只有一个ClassLoader，也就是SystemClassLoader
* 在以fat jar运行时，有一个`LaunchedURLClassLoader`，它的parent是SystemClassLoader

    `LaunchedURLClassLoader`的urls是fat jar里的`BOOT-INF/classes`和`BOOT-INF/lib`下的jar。SystemClassLoader的urls是fat jar本身。

* 在解压目录（exploded directory）运行时，和fat jar类似，不过url都是目录形式。目录形式会有更好的兼容性。

### spring boot 1.3.* 和 1.4.* 版本的区别

在spring boot 1.3.* 版本里

* 应用的类和spring boot loader的类都是打包在一个fat jar里
* 应用依赖的jar放在fat jar里的`/lib`下面。

在spring boot 1.4.* 版本后

* spring boot loader的类放在fat jar里
* 应用的类打包放在fat jar的`BOOT-INF/classes`目录里
* 应用依赖的jar放在fat jar里的`/lib`下面。

spring boot 1.4的打包结构改动是这个commit引入的
https://github.com/spring-projects/spring-boot/commit/87fe0b2adeef85c842c009bfeebac1c84af8a5d7

这个commit的本意是简化classloader的继承关系，以一种直观的parent优先的方式来实现`LaunchedURLClassLoader`，同时打包结构和传统的war包应用更接近。

但是这个改动引起了很多复杂的问题，从上面我们分析的ClassLoader继承关系就有点头晕了。


## 目前的ClassLoader继承关系带来的一些影响

有很多用户可能会发现，一些代码在IDE里跑得很好，但是在实际部署运行时不工作。很多时候就是ClassLoader的结构引起的，下面分析一些案例。

### `demo.jar!/BOOT-INF/classes!/` 这样子url不工作

因为spring boot是扩展了标准的jar协议，让它支持多层的jar in jar，还有directory in jar。参考[spring boot应用启动原理分析](http://hengyunabc.github.io/spring-boot-application-start-analysis/)

在spring boot 1.3的时候尽管会有jar in jar，但是一些比较健壮的代码可以处理这种情况，比如tomcat8自己就支持jar in jar。

但是绝大部分代码都不会支持像`demo.jar!/BOOT-INF/classes!/` 这样子directory in jar的多重url，所以在spring boot1.4里，很多库的代码都会失效。

### `demo.jar!/META-INF/resources` 下的资源问题

在servlet 3.0规范里，应用可以把静态资源放在`META-INF/resources`下面，servlet container会支持读取。但是从上面的继承结果，我们可以发现一个问题：

* 应用以fat jar来启动，启动embedded tomcat的ClassLoader是`LaunchedURLClassLoader`
* `LaunchedURLClassLoader`的urls并没有fat jar本身
* 应用的main函数所在的模块的`src/main/resources/META-INF/resources`目录被打包到了fat jar里，也就是`demo.jar!/META-INF/resources`
* 应用的fat jar是SystemClassLoader的url，也就是`LaunchedURLClassLoader`的parent

这样子就造成了一些奇怪的现象：

* 应用直接用自己的ClassLoader.getResources()是可以获取到`META-INF/resources`的资源的
* 但是embedded tomcat并没有把fat jar本身加入到它的 ResourcesSet 里，因为它在启动时ClassLoader是`LaunchedURLClassLoader`，它只扫描自己的ClassLoader的`urls`
* 应用把资源放在其它的jar包的`META-INF/resources`下可以访问到，把资源放在自己的main函数的`src/main/resources/META-INF/resources`下时，访问不到了

另外，spring boot的官方jsp的例子只支持war的打包格式，不支持fat jar，也是由这个引起的。


### `getResource("")` 和 `getResources("")` 的返回值的问题


`getResource("")`的语义是返回ClassLoader的urls的第一个url，很多时候使用者以为这个就是它们自己的classes的目录，或者是jar的url。

但是实际上，因为ClassLoader加载urls列表时，有随机性，和OS低层实现有关，并不能保证urls的顺序都是一样的。所以`getResource("")`很多时候返回的结果并不一样。

但是很多库，或者应用依赖这个代码来定位扫描资源，这样子在spring boot下就不工作了。

另外，值得注意的是spring boot在三种不同形式下运行，`getResources("")`返回的结果也不一样。用户可以自己改下demo里的代码，打印下结果。

简而言之，不要依赖这两个API，最好自己放一个资源来定位。或者直接利用spring自身提供的资源扫描机制。


### 类似 `classpath*:**-service.xml` 的通配问题

用户有多个代码模块，在不同模块下都放了多个`*-service.xml`的spring配置文件。

用户如果使用类似`classpath*:**-service.xml`的通配符来加载资源的话，很有可能出现在IDE里跑时，可以正确加载，但是在fat jar下，却加载不到的问题。


从spring自己的文档可以看到相关的解析：

https://docs.spring.io/spring/docs/4.3.9.RELEASE/javadoc-api/org/springframework/core/io/support/PathMatchingResourcePatternResolver.html

> WARNING: Note that "classpath*:" when combined with Ant-style patterns will only work reliably with at least one root directory before the pattern starts, unless the actual target files reside in the file system. This means that a pattern like "classpath*:*.xml" will not retrieve files from the root of jar files but rather only from the root of expanded directories. This originates from a limitation in the JDK's ClassLoader.getResources() method which only returns file system locations for a passed-in empty String (indicating potential roots to search). This ResourcePatternResolver implementation is trying to mitigate the jar root lookup limitation through URLClassLoader introspection and "java.class.path" manifest evaluation; however, without portability guarantees.


就是说使用 `classpath*`来匹配其它的jar包时，需要有一层目录在前面，不然的话是匹配不到的，这个是ClassLoader.getResources() 函数导致的。

因为在IDE里跑时，应用所依赖的其它模块通常就是一个`classes`目录，所以通常没有问题。

但是当以fat jar来跑时，其它的模块都被打包为一个jar，放在`BOOT-INF/lib`下面，所以这时通配就会失败。


## 总结

* 这个新的`BOOT-INF`打包格式有它的明显好处：更清晰，更接近war包的格式。
* spring boot的ClassLoader的结构修改带来的复杂问题，并非当初修改的人所能预见的
* 很多问题需要理解spring boot的ClassLoader结构，否则不能找到根本原因
