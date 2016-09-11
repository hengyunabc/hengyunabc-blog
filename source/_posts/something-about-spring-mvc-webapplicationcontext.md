title: 扯谈spring mvc之WebApplicationContext的继承关系
date: 2015-07-26 17:51:27
tags:
 - spring
 - springmvc
 - mvc
 - java

categories:
 - 技术


---

## spring mvc里的root/child WebApplicationContext的继承关系

在传统的spring mvc程序里会有两个````WebApplicationContext````，一个是parent，从````applicationContext.xml````里加载的，一个是child，从````servlet-context.xml````里加载的。
两者是继承关系，child WebApplicationContext 可以通过````getParent()````函数获取到root WebApplicationContext。

简单地说child WebApplicationContext里的bean可以注入root WebApplicationContext里的bean，而parent WebApplicationContext的bean则不能注入chile WebApplicationContext里的bean。


一个典型的web.xml的内容是：

```xml
	<!-- The definition of the Root Spring Container shared by all Servlets and Filters -->
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>classpath*:/applicationContext.xml</param-value>
	</context-param>
	
	<!-- Creates the Spring Container shared by all Servlets and Filters -->
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

	<!-- Processes application requests -->
	<servlet>
		<servlet-name>appServlet</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/servlet-context.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
		<async-supported>true</async-supported>
	</servlet>
		
	<servlet-mapping>
		<servlet-name>appServlet</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>
```

其中root WebApplicationContext是通过listener初始化的，child WebApplicationContext是通过servlet初始化的。

而在````applicationContext.xml````里通常只component-scan非Controller的类，如：

```xml
	<context:component-scan base-package="io.github.test">
		<context:exclude-filter expression="org.springframework.stereotype.Controller"
			type="annotation" />
		<context:exclude-filter type="annotation"
			expression="org.springframework.web.bind.annotation.ControllerAdvice" />
	</context:component-scan>
```

在````servlet-context.xml````里通常只component-scan Controller类，如：

```xml
	<context:component-scan base-package="io.github.test.web" use-default-filters="false">
		<context:include-filter expression="org.springframework.stereotype.Controller"
			type="annotation" />
		<context:include-filter type="annotation"
			expression="org.springframework.web.bind.annotation.ControllerAdvice" />
	</context:component-scan>
```
如果不这样子分别component-scan的话，可能会出现Bean重复初始化的问题。

上面是Spring官方开始时推荐的做法。

## root/child WebApplicationContext继承关系带来的麻烦

root WebApplicationContext里的bean可以在不同的child WebApplicationContext里共享，而不同的child WebApplicationContext里的bean区不干扰，这个本来是个很好的设计。

但是实际上有会不少的问题：
* 不少开发者不知道Spring mvc里分有两个WebApplicationContext，导致各种重复构造bean，各种bean无法注入的问题。
* 有一些bean，比如全局的aop处理的类，如果先root WebApplicationContext里初始化了，那么child WebApplicationContext里的初始化的bean就没有处理到。如果在chile WebApplicationContext里初始化，在root WebApplicationContext里的类就没有办法注入了。
* 区分哪些bean放在root/child很麻烦，不小心容易搞错，而且费心思。

## 一劳永逸的解决办法：bean都由root WebApplicationContext加载
在一次配置[metrics-spring](https://github.com/ryantenney/metrics-spring "")时，对配置````@EnableMetrics````配置在哪个WebApplicationContext里，感到很蛋疼。最终决定试下把所有的bean，包括Controller都移到root WebApplicationContext，即````applicationContext.xml````里加载，而````servlet-context.xml````里基本是空的。结果发现程序运行完全没问题。

后面在网上搜索了下，发现有一些相关的讨论：

http://forum.spring.io/forum/spring-projects/container/89149-servlet-context-vs-application-context 


## spring boot里的做法

在spring boot里默认情况下不需要component-scan的配置，于是猜测在Spring boot里是不是只有一个WebApplicationContext？

后面测试下了，发现在spring boot里默认情况下的确是只有一个WebApplicationContext：````org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext````，所以在spring boot里省事了很多。

## 总结
spring 的ApplicationContext继承机制是一个很好的设计，在很多其它地方都可以看到类似的思路，比如Java的class loader。但是在大部分spring web程序里，实际上只要一个WebApplicationContext就够了。如果分开rott/child WebApplicationContext会导致混乱，而没什么用。

所以推荐把所有的Service/Controller都移到root WebApplicationContext中初始化。

