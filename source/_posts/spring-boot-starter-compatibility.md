---
title: 深入Spring Boot：编写兼容Spring Boot1和Spring Boot2的Starter
date: 2018-07-25 19:58:28
tags:
 - spring
 - spring-boot
 - java
 - asm

categories:
  - 技术

---


## 前言


Spring Boot 2正式发布已经有段时间，应用升级之前，starter先要升级，那么如何支持Spring Boot 2？

## 为什么选择starter同时兼容spring boot 1和spring boot 2

* 从用户角度来看

    如果不在一个starter里兼容，比如用版本号来区分，spring boot 1的用户使用`1.*`，spring boot 2用户使用`2.*`，这样用户升级会有很大困扰。

    另外，我们的starter是以日期为版本号的，如果再分化，则就会出现`2018-06-stable-boot1`，`2018-06-stable-boot2`，这样子很丑陋。

* 从开发者角度来看

    要同时维护两个分支，修改代码时要合到两个分支上，发版本时要同时两个。如果有统一的bom文件，也需要维护两份。工作量翻倍，而且很容易出错。

因此，我们决定在同一个代码分支里，同时支持spring boot 1/2。减少开发维护成本，减少用户使用困扰。

## 编写兼容的starter的难点

spring boot starter的代码入口都是在各种`@Configuration`类里，这为我们编写兼容starter提供了条件。

但还是有一些难点：

* 某些类不兼容，比如在spring boot 2里删除掉了
* 代码模块，maven依赖怎样组织
* 怎样保证starter在spring boot 1/2里都能正常工作

## 通过ASM分析现有的starter里不兼容的类

* https://github.com/hengyunabc/springboot-classchecker

springboot-classchecker可以从jar包里扫描出哪些类在spring boot 2里不存在的。

> 工作原理：springboot-classchecker自身在pom.xml里依赖的是spring boot 2，扫描jar包里通过ASM分析到所有的String，提取出类名之后，再尝试在ClassLoader里加载，如果加载不到，则说明这个类在spring boot 2里不存在。

例如扫描`demo-springboot1-starter.jar` ：

```bash
mvn clean package
java -jar target/classchecker-0.0.1-SNAPSHOT.jar demo-springboot1-starter.jar
```

结果是：

```bash
path: demo-springboot1-starter.jar
org.springframework.boot.actuate.autoconfigure.ConditionalOnEnabledHealthIndicator
org.springframework.boot.actuate.autoconfigure.EndpointAutoConfiguration
org.springframework.boot.actuate.autoconfigure.HealthIndicatorAutoConfiguration
```

那么这些类在spring boot 2在哪里了？

实际上是改了package：

```
org.springframework.boot.actuate.autoconfigure.health.ConditionalOnEnabledHealthIndicator
org.springframework.boot.actuate.autoconfigure.endpoint.EndpointAutoConfiguration
org.springframework.boot.actuate.autoconfigure.health.HealthIndicatorAutoConfiguration
```

通过扫描20多个starter jar包，发现不兼容的类有：

* org.springframework.boot.env.PropertySourcesLoader
* org.springframework.boot.autoconfigure.jdbc.DataSourceBuilder
* org.springframework.boot.bind.RelaxedDataBinder
* Endpoint/HealthIndicator 相关的类

可以总结：

* spring boot核心的类，autoconfigure相关的没有改动
* 大部分修改的是Endpoint/HealthIndicator 相关的类

## spring-boot-utils兼容工具类

* https://github.com/hengyunabc/spring-boot-utils

spring-boot-utils提供兼容工具类，同时支持spring boot 1/2。

### BinderUtils

在spring boot 1里，注入环境变量有时需要用到`RelaxedDataBinder`：

```java
MyProperties myProperties = new MyProperties();
MutablePropertySources propertySources = environment.getPropertySources();
new RelaxedDataBinder(myProperties, "spring.my").bind(new PropertySourcesPropertyValues(propertySources));
```

在spring boot 2里，`RelaxedDataBinder`删除掉了，新的写法是用`Binder`：

```java
Binder binder = Binder.get(environment);
MyProperties myProperties = binder.bind("spring.my", MyProperties.class).get();
```

通过BinderUtils，则可以同时支持spring boot1/2：

```java
MyProperties myProperties = BinderUtils.bind(environment, "spring.my", MyProperties.class);
```

### @ConditionalOnSpringBoot1/@ConditionalOnSpringBoot2

spring boot starter的功能大部分都是通过`@Configuration`组装起来的。spring boot 1的Configuration类，不能在spring boot 2里启用。则可以通过`@ConditionalOnSpringBoot1`，`@ConditionalOnSpringBoot2`这两个注解来分别支持。

其实原理很简单，判断spring boot 1/2里各自有的存在的类就可以了。

```java
@ConditionalOnClass(name = "org.springframework.boot.bind.RelaxedDataBinder")
public @interface ConditionalOnSpringBoot1 {
}
```

```java
@ConditionalOnClass(name = "org.springframework.boot.context.properties.bind.Binder")
public @interface ConditionalOnSpringBoot2 {
}
```


## Starter代码模块组织

下面以实际的一个starter来说明。

* https://github.com/hengyunabc/endpoints-spring-boot-starter

> spring boot web应用的mappings信息，可以在`/mappings` endpoint查询到。但是这么多endpoint，它们都提供了哪些url？

> endpoints-spring-boot-starter的功能是展示所有endpoints的url mappings信息


`endpoints-spring-boot-starter`里需要给spring boot 1/2同时提供endpoint功能，代码模块如下：


```
endpoints-spring-boot-starter
|__ endpoints-spring-boot-autoconfigure1
|__ endpoints-spring-boot-autoconfigure2
```

* endpoints-spring-boot-autoconfigure1模块在pom.xml里依赖的是spring boot 1相关的jar，并且都设置为`<optional>true</optional>`
* endpoints-spring-boot-autoconfigure2的配置类似
* endpoints-spring-boot-starter依赖autoconfigure1 和 autoconfigure2
* 如果有公共的逻辑，可以增加一个commons模块

### Endpoint兼容

以 endpoints-spring-boot-autoconfigure1模块为例说明怎样处理。

* `EndPointsEndPoint`类继承自spring boot 1的`AbstractMvcEndpoint`：

  ```java
  @ConfigurationProperties("endpoints.endpoints")
  public class EndPointsEndPoint extends AbstractMvcEndpoint {
  ```

* 通过`@ManagementContextConfiguration`引入

  ```java
  @ManagementContextConfiguration
  public class EndPointsEndPointManagementContextConfiguration {

      @Bean
      @ConditionalOnMissingBean
      @ConditionalOnEnabledEndpoint("endpoints")
      public EndPointsEndPoint EndPointsEndPoint() {
          EndPointsEndPoint endPointsEndPoint = new EndPointsEndPoint();
          return endPointsEndPoint;
      }

  }
  ```

* 在`META-INF/resources/spring.factories`里配置

  ```
  org.springframework.boot.actuate.autoconfigure.ManagementContextConfiguration=\
  io.github.hengyunabc.endpoints.autoconfigure1.EndPointsEndPointManagementContextConfiguration
  ```

因为`org.springframework.boot.actuate.autoconfigure.ManagementContextConfiguration`是只在spring boot 1里，在spring boot 2的应用里不会加载它，所以autoconfigure1模块天然兼容spring boot 2。

那么类似的，autoconfigure2模块里在`META-INF/resources/spring.factories`配置的是

```
org.springframework.boot.actuate.autoconfigure.web.ManagementContextConfiguration=\
io.github.hengyunabc.endpoints.autoconfigure2.ManagementApplicationcontextHolderConfiguration
```

**仔细对比，可以发现是spring boot 2下面修改了`ManagementContextConfiguration`的包名，所以对于Endpoint天然是兼容的，不同的模块自己编绎就可以了。**


## HealthIndicator的兼容

类似Endpoint的处理，spring boot 1/2的代码分别放不同的autoconfigure模块里，然后各自的`@Configuration`类分别使用`@ConditionalOnSpringBoot1/@ConditionalOnSpringBoot2`来判断。


## 通过集成测试保证兼容性

还是以endpoints-spring-boot-autoconfigure1模块为例。

这个模块是为spring boot 1准备的，则它的集成测试要配置为spring boot 2。

参考相关的代码：[查看](https://github.com/hengyunabc/endpoints-spring-boot-starter/tree/endpoints-spring-boot-starter-parent-0.0.1/endpoints-spring-boot-autoconfigure1/src/it)

* 在`springboot2demo/pom.xml`里依赖spring boot 2
* 在`verify.groovy`里检测应用是否启动成功

## 总结

* 通过ASM分析现有的starter里不兼容的类
* 配置注入通过`BinderUtils`解决
* 各自的`@Configuration`类分别用`@ConditionalOnSpringBoot1/@ConditionalOnSpringBoot2`来判断
* 代码分模块：commons放公共逻辑, autoconfigure1/autoconfigure2 对应 spring boot 1/2的自动装配，starter给应用依赖
* Endpoint的Configuration入口是ManagementContextConfiguration，因为spring boot 2里修改了package，所以直接在`spring.factories`里配置即可
* 通过集成测试保证兼容性
* 如果某一天，不再需要支持spring boot 1了，则直接把autoconfigure1模块去掉即可

## 链接 

* https://github.com/hengyunabc/spring-boot-utils 
* https://github.com/hengyunabc/springboot-classchecker
* https://github.com/hengyunabc/endpoints-spring-boot-starter
