---
title: 为什么在Spring的配置里，最好不要配置xsd文件的版本号
date: 2014-03-30 05:58:28
tags:
 - spring
 - java
 - xml


categories:
  - 技术

---


## 为什么dubbo启动没有问题？

这篇blog源于一个疑问：

我们公司使了阿里的dubbo，但是阿里的开源网站http://code.alibabatech.com，挂掉有好几个月了，为什么我们的应用启动没有问题？

我们的应用的Spring配置文件里有类似的配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
	        http://www.springframework.org/schema/beans/spring-beans.xsd
	        http://code.alibabatech.com/schema/dubbo
	        http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
```

我们都知道Spring在启动时是要检验XML文件的。或者为什么在Eclipse里xml没有错误提示？
比如这样的一个Spring配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
</beans>
```

我们也可以在后面加上版本号：

```xml
xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
```

**有这个版本号和没有有什么区别呢？**

## XML的一些概念

首先来看下xml的一些概念：

xml的schema里有namespace，可以给它起个别名。比如常见的spring的namespace：

```xml
	xmlns:mvc="http://www.springframework.org/schema/mvc"
	xmlns:context="http://www.springframework.org/schema/context"
```

通常情况下，namespace对应的URI是一个存放XSD的地址，尽管规范没有这么要求。如果没有提供schemaLocation，那么Spring的XML解析器会从namespace的URI里加载XSD文件。我们可以把配置文件改成这个样子，也是可以正常工作的：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans/spring-beans.xsd"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
```

schemaLocation提供了一个xml namespace到对应的XSD文件的一个映射，所以我们可以看到，在xsi:schemaLocation后面配置的字符串都是成对的，前面的是namespace的URI，后面是xsd文件的URI。比如：

```xml
	xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/security
	http://www.springframework.org/schema/security/spring-security.xsd"
```

## Spring是如何校验XML的

Spring默认在启动时是要加载XSD文件来验证xml文件的，所以如果有的时候断网了，或者一些开源软件切换域名，那么就很容易碰到应用启动不了。我记得当时Oracle收购Sun公司时，遇到过这个情况。

为了防止这种情况，Spring提供了一种机制，默认从本地加载XSD文件。打开spring-context-3.2.0.RELEASE.jar，可以看到里面有两个特别的文件：

* spring.handlers

	```xml
	http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
	http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler
	http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler
	http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler
	http\://www.springframework.org/schema/cache=org.springframework.cache.config.CacheNamespaceHandler
	```

* spring.schemas

	```
	http\://www.springframework.org/schema/context/spring-context-2.5.xsd=org/springframework/context/config/spring-context-2.5.xsd
	http\://www.springframework.org/schema/context/spring-context-3.0.xsd=org/springframework/context/config/spring-context-3.0.xsd
	http\://www.springframework.org/schema/context/spring-context-3.1.xsd=org/springframework/context/config/spring-context-3.1.xsd
	http\://www.springframework.org/schema/context/spring-context-3.2.xsd=org/springframework/context/config/spring-context-3.2.xsd
	http\://www.springframework.org/schema/context/spring-context.xsd=org/springframework/context/config/spring-context-3.2.xsd
	...
	```

再打开jar包里的org/springframework/context/config/ 目录，可以看到下面有

```
spring-context-2.5.xsd
spring-context-3.0.xsd
spring-context-3.1.xsd
spring-context-3.2.xsd
```

很明显，可以想到Spring是把XSD文件放到本地了，再在spring.schemas里做了一个映射，优先从本地里加载XSD文件。

并且Spring很贴心，把旧版本的XSD文件也全放了。这样可以防止升级了Spring版本，而配置文件里用的还是旧版本的XSD文件，然后断网了，应用启动不了。

我们还可以看到，在没有配置版本号时，用的就是当前版本的XSD文件：

```xml
http\://www.springframework.org/schema/context/spring-context.xsd=org/springframework/context/config/spring-context-3.2.xsd
```


同样，我们打开dubbo的jar包，可以在它的spring.schemas文件里看到有这样的配置：

```
http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
```

所以，Spring在加载dubbo时，会从dubbo的jar里加载dubbo.xsd。

## 如何跳过Spring的XML校验？

可以用这样的方式来跳过校验：

```java
GenericXmlApplicationContext context = new GenericXmlApplicationContext();
context.setValidating(false);
```

## 如何写一个自己的spring xml namespace扩展

可以参考Spring的文档，实际上是相当简单的。只要实现自己的NamespaceHandler，再配置一下spring.handlers和spring.schemas就可以了。

http://docs.spring.io/spring/docs/current/spring-framework-reference/html/extensible-xml.html

## 其它的一些东东

* 防止XSD加载不成功的一个思路

    http://hellojava.info/?p=135

* 齐全的Spring的namespace的列表

    http://stackoverflow.com/questions/11174286/spring-xml-namespaces-how-do-i-find-what-are-the-implementing-classes-behind-t

* Spring core

	```
	aop - AopNamespaceHandler
	c - SimpleConstructorNamespaceHandler
	cache - CacheNamespaceHandler
	context - ContextNamespaceHandler
	jdbc - JdbcNamespaceHandler
	jee - JeeNamespaceHandler
	jms - JmsNamespaceHandler
	lang - LangNamespaceHandler
	mvc - MvcNamespaceHandler
	oxm - OxmNamespaceHandler
	p - SimplePropertyNamespaceHandler
	task - TaskNamespaceHandler
	tx - TxNamespaceHandler
	util - UtilNamespaceHandler
	```

* Spring Security

	```
	security - SecurityNamespaceHandler
	oauth - OAuthSecurityNamespaceHandler
	```

* Spring integration

	```
	int - IntegrationNamespaceHandler
	amqp - AmqpNamespaceHandler
	event - EventNamespaceHandler
	feed - FeedNamespaceHandler
	file - FileNamespaceHandler
	ftp - FtpNamespaceHandler
	gemfire - GemfireIntegrationNamespaceHandler
	groovy - GroovyNamespaceHandler
	http - HttpNamespaceHandler
	ip - IpNamespaceHandler
	jdbc - JdbcNamespaceHandler
	jms - JmsNamespaceHandler
	jmx - JmxNamespaceHandler
	mail - MailNamespaceHandler
	redis - RedisNamespaceHandler
	rmi - RmiNamespaceHandler
	script - ScriptNamespaceHandler
	security - IntegrationSecurityNamespaceHandler
	sftp - SftpNamespaceHandler
	stream - StreamNamespaceHandler
	twitter - TwitterNamespaceHandler
	ws - WsNamespaceHandler
	xml - IntegrationXmlNamespaceHandler
	xmpp - XmppNamespaceHandler
	```

## 总结

为什么不要在Spring的配置里，配置上XSD的版本号？
因为如果没有配置版本号，取的就是当前jar里的XSD文件，减少了各种风险。
而且这样约定大于配置的方式很优雅。

## 参考

http://stackoverflow.com/questions/10768873/spring-di-applicationcontext-xml-how-exactly-is-xsischemalocation-used

http://stackoverflow.com/questions/11174286/spring-xml-namespaces-how-do-i-find-what-are-the-implementing-classes-behind-t

http://docs.spring.io/spring/docs/current/spring-framework-reference/html/extensible-xml.html