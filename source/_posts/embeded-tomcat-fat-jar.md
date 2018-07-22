---
title: 应用内置embeded tomcat，并打包为fat jar的解决方案
date: 2016-04-05 21:58:28
tags:
 - java
 - tomcat
 - fatjar


categories:
  - 技术

---



## 需求

大量的微服务框架引起了一大波embeded tomcat，executable fat jar的潮流。显然spring boot是最出色的解决方案，但是spring boot有两个不足的地方：

* 不支持配置web.xml文件，对于旧应用迁移不方便
* 一些配置在web.xml里配置起来很直观，放到代码里配置就难搞清楚顺序了。比如一些filter的顺序关系。
* spring boot的方案依赖spring，对于一些轻量级的应用不想引入依赖

基于这些考虑，这里提出一个基于embeded tomcat本身的解决方案。

## 代码地址

https://github.com/hengyunabc/executable-embeded-tomcat-sample

支持特性：

* 支持加载传统的web.xml配置
* 支持打包为fat jar方式运行
* 支持在IDE里直接运行

## 旧应用迁移步聚
旧应用迁移非常的简单

* 在pom.xml里增加embeded tomcat的依赖
* 把应用原来的`src/main/webapp/WEB-INF` 移动到 `src/main/resources/WEB-INF`下，把在`src/main/webapp`下面的所有文件移动到 `src/main/META-INF/resources`目录下
* 写一个main函数，把tomcat启动起来

非常的简单，完全支持旧应用的所有功能，不用做任何的代码改造。

## 工作原理

### web.xml的读取

传统的Tomcat有两个目录，一个是baseDir，对应tomcat本身的目录，下面有conf, bin这些文件夹。一个是webapp的docBase目录，比如webapps/ROOT 这个目录。

docBase只能是一个目录，或者是一个`.war`结尾的文件（实际对应不解压war包运行的方式）。

tomcat里的一个webapp对应有一个Context，Context里有一个`WebResourceRoot`，应用里的资源都是从`WebResourceRoot` 里加载的。tomcat在初始化Context时，会把docBase目录加到`WebResourceRoot`里。

tomcat在加载应用的web.xml里，是通过`ServletContext`来加载的，而`ServletContext`实际上是通过`WebResourceRoot`来获取到资源的。

所以简而言之，需要在tomcat启动之前，web.xml放到Context的`WebResourceRoot`，这样子tomcat就可以读取到web.xml里。

### 静态资源的处理

在Servlet 3.0规范里，应用可以把静态的资源放在jar包里的`/META-INF/classes`目录，或者在`WEB-INF/classes/META-INF/resources`目录下。

所以，采取了把资源文件全都放到`src/main/META-INF/resources`目录的做法，这样子有天然符合Servlet的规范，而且在打包时，自然地打包到fat jar里。

### Fat jar的支持
Fat jar的支持是通过`spring-boot-maven-plugin`来实现的，它提供了把应用打包为fat jar，并启动的能力。具体原理可以参考另外一篇博客：http://blog.csdn.net/hengyunabc/article/details/50120001

当然，也可以用其它的方式把依赖都打包到一起，比如`maven-assembly-plugin/jar-with-dependencies` ，但不推荐，毕竟spring boot的方案很优雅。

## 其它

http://home.apache.org/~markt/presentations/2010-11-04-Embedding-Tomcat.pdf   

官方的Embedded Tomcat文档

http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/basic_app_embedded_tomcat/basic_app-tomcat-embedded.html

oracle的技术文档里的一个方案，但是这个方案太简单了，不支持在IDE里启动，不支持加载web.xml文件。


https://github.com/kui/executable-war-sample

这个把依赖都打包进war包里，然后在初始化tomcat里，直接把这个war做为docBase。这样子可以加载到web.xml。但是有一个严重的安全问题，因为应用的`.class`文件直接在war的根目录下，而不是在`/WEB-INF/classes`目录下，所以可以直接通过http访问到应用的`.class`文件，即攻击者可以直接拿到应用的代码来逆向分析。这个方案并不推荐使用。

实际上spring boot应用以一个war包直接运行时，也是有这个安全问题的。只是spring boot泄露的只是spring boot loader的`.class`文件。

