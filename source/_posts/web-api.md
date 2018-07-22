---
title: Web API 版本控制的几种方式
date: 2014-03-05 05:58:28
tags:
 - web
 - restful


categories:
  - 技术

---




http://www.troyhunt.com/2014/02/your-api-versioning-is-wrong-which-is.html

这篇文章写得很好，介绍了三种实现web api版本化的三种方式。我从评论里又收集到两种方式，所以一共是5种：

方式一：利用URL

```
HTTP GET:
https://haveibeenpwned.com/api/v2/breachedaccount/foo
```

方式二：利用用户自定义的request header

```
HTTP GET:
https://haveibeenpwned.com/api/breachedaccount/foo
api-version: 2
```

方式三：利用content type

```
HTTP GET:
https://haveibeenpwned.com/api/breachedaccount/foo
Accept: application/vnd.haveibeenpwned.v2+json
```

方式四：利用content type

```
HTTP GET:
https://haveibeenpwned.com/api/breachedaccount/foo
Accept: application/vnd.haveibeenpwned+json; version=2.0
```

这个方式和方式三的小不同的地方是，把版本号分离出来了。

方式五：利用URL里的parameter

```
HTTP GET:
https://haveibeenpwned.com/api/breachedaccount/foo?v=2
```

作者说他最喜欢第三种方式，因为

1. URL不用改变
1. 客户端应该通过accept header来表明自己想接收的是什么样的数据。

但作者很蛋疼地在他的网站上把前面三种方式都实现了，而且都支持。
https://haveibeenpwned.com/API/v2

我个人最喜欢的是第二种方式，因为这个用spring mvc实现最容易，也最简洁。

因为只要在Controler上用@RequestMapping标明版本即可。不用再去各种匹配，各种识别。

如果是自己写一个Annotation来识别的话，也要花些功夫，而且怎么无缝地转发到原有的Spring mvc的配置也是个问题。

```java
@Controller
@RequestMapping(headers="apt-version=2")
public class TestControllerV2 {
}
```

另外这个网站列举了很多国外的有名网站是如何实现web api版本控制的。

http://www.lexicalscope.com/blog/2012/03/12/how-are-rest-apis-versioned/

