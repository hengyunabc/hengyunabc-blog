---
title: 扯谈web安全之JSON
date: 2014-05-29 16:58:28
tags:
 - web
 - json
 - security
 - javascript


categories:
  - 技术

---

## 前言

[JSON](http://www.json.org/)（JavaScript Object Notation），可以说是事实的浏览器，服务器交换数据的标准了。目测其它的格式如XML，或者其它自定义的格式会越来越少。

为什么JSON这么流行？
和JavaScript无缝对接是一个原因。

还有一个重要原因是可以比较轻松的实现跨域。如果是XML，或者其它专有格式，则很难实现跨域，要通过flash之类来实现。


任何一种数据格式，如何解析处理不当，都会存在安全漏洞。下面扯谈下JSON相关的一些安全东东。
在介绍之前，先来提几个问题：

* 为什么XMLHttpRequest要遵守同源策略？
* XMLHttpRequest 请求会不会带cookie?
* `<script scr="...">` 的标签请求会不会带cookie？
* 向一个其它域名的网站提交一个form，会不会带cookie？
* CORS请求能不能带cookie？

## JSON注入

有的时候，可能是为了方便，有人会手动拼接下JSON，但是这种随手代码，却可能带来意想不到的安全隐患。

### 利用字符串拼接

第一种方式，利用字符串拼接：

```java
		String user = "test01";
		String password = "12345', admin:'true";
		String json = "{user:'%s', password:'%s'}";
		System.out.println(String.format(json, user, password));
		//{user:'test01', password:'12345', admin:'true'}
```

用户增加了管理员权限。

### 利用Parameter pollution， 类似http parameter pollution

第二种，利用Parameter pollution， 类似http parameter pollution


```java
		String string = "{user:'test01',password:'hello', password:'world'}";
		JSONObject parse = JSON.parseObject(string);
		String password = parse.getString("password");
		System.out.println(password);
		//world
```

当JSON数据key重复了会怎么处理？大部分JSON解析库都是后面的参数覆盖了前面的。

下面的演示了修改别人密码的例子：

```java
		//user%3Dtest01%26password%3D12345%27%2Cuser%3Dtest02"
		//user=test01&password=12345',user=test02
		HttpServletRequest request = null;
		String user = request.getParameter("user");
		//检查test01是否登陆
		String password = request.getParameter("password");
		String content = "{user:'" + user + "', password:'" + password + "'}";
		 
		User user = JSON.parseObject(content, User.class);
		//{"password":"12345","user":"test02"}
		updateDb(user);
```

**所以说，不要手动拼接JSON字符串**

## 浏览器端应该如何处理JSON数据？

像eval这种方式，自然是不能采用的。

现在的浏览器都提供了原生的方法 JSON.parse(str) 来转换为JS对象。

如果是IE8之前的浏览器，要使用这个库来解析：https://github.com/douglascrockford/JSON-js
参考：http://zh.wikipedia.org/wiki/JSON#.E5.AE.89.E5.85.A8.E6.80.A7.E5.95.8F.E9.A1.8C

JQuery里内置了JSON解析库

## JSONP callback注入



### jsonp工作原理

简单介绍jsonp的工作原理。

用js在document上插入一个<`script>`标签，标签的src指向远程服务器的API地址。客户端和服务器约定一个回调函数的名字，然后服务器返回回调函数包裹着的数据，然后浏览器执行回调函数，取得数据。

比如jquery是这样子实现的：

```javascript
var url = 'http://localhost:8080/testJsonp?callback=?';
$.getJSON(url, function(data){
    alert(data)
});
```

jquery自动把?转成了一个带时间戳特别的函数（防止缓存）：

```
  http://localhost:8080/testJsonp?callback=jQuery1102045087050669826567_1386230674292&_=1386230674293

```

相当于插入了这么一个`<script>`标签：

```html
<script src="http://localhost:8080/testJsonp?callback=jQuery1102045087050669826567_1386230674292&_=1386230674293"></script>
```

服务器返回的数据是这样子的：

```json
jQuery1102045087050669826567_1386230674292({'name':'abc', 'age':18})
```

浏览器会执行直接执行这个JS函数。

**所以，如果在callback函数的名字上做点手脚，可以执行任意的JS代码。所以说callback名字一定要严格过滤。**

当然，callback函数的名字通常是程序自己控制的，但是不能排除有其它被利用的可能。
那么callback函数的名字，如何过滤？应当只允许合法的JS函数命名，用正则来匹配应该是这样子的：

```
^[0-9a-zA-Z_.]+$
```

正则可能比较慢，可以写一个函数来判断：

```java
	static boolean checkJSONPCallbackName(String name) {
		try {
			for (byte c : name.getBytes("US-ASCII")) {
				if ((c >= '0' && c <= '9') || (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || c == '_')
				{
					continue;
				} else {
					return false;
				}
			}
			return true;
		} catch (Throwable t) {
			return false;
		}
	}
```

实际上对callback函数名字进行严格检验还有其它的一个好处，就是防范了很多UTF-7编码攻击。

因为UTF-7编码的头部都是带有特殊字符的，如"+/v8"，"+/v9"，这样就过滤掉非法编码的请求了。

update: 2016-3-22 

spring 4.1里有一个AbstractJsonpResponseBodyAdvice ，里面是用正则匹配来判断是否合法的jsonp函数名。

```java
public abstract class AbstractJsonpResponseBodyAdvice extends AbstractMappingJacksonResponseBodyAdvice {
 
	/**
	 * Pattern for validating jsonp callback parameter values.
	 */
	private static final Pattern CALLBACK_PARAM_PATTERN = Pattern.compile("[0-9A-Za-z_\\.]*");
```

## jsonp的请求的验证

jsonp在使用的时候，还有容易犯的错误是没有验证用户的身份。

**第一，操作是否是用户自己提交的，而不是别的网页用`<script>`标签，或者用`<form>`提交的**

所以要检查request的refer，或者验证token。这个实际是CSRF防护的范畴，但是很容易被忽略。

**第二，要验证用户的权限。**

很多时候，可能只是验证了用户是否登录 。却没有仔细判断用户是否有权限。

比如通过JSONP请求修改了别的用户的数据。
所以说，一定要难证来源。判断refer，验证用户的身份。

到乌云上搜索，可以找到不少类似的漏洞，都是因为没有严格验证用户的权限。http://www.wooyun.org/searchbug.php?q=jsonp

## JSON hijacking

在JS里可以为对象定义一些setter函数，这样的话就存在了可以利用的漏洞。
比如在浏览器的JS Console里执行：

```js
window.__defineSetter__('x', function() {
alert('x is being assigned!');
});
window.x=1;
```

会很神奇地弹出一个alert窗口，说明我们定义的setter函数起作用了。
结合这个，当利用`<script>`标签请求外部的一个JSON API时，如果返回的是数组型，就可以利用窃取数据。

比如有这样的一个API：
http://www.test.com/friends

返回的数据是JSON Array：

```json
[{'user':'test01','age':18},{'user':'test02,'age':19},{'user':'test03','age':20}]
```

在攻击页面上插入以下的代码，就可以获取到用户的所有的朋友的信息。

```html
<script>
  Object.prototype.__defineSetter__('user',function(obj)
      {alert(obj);
    }
  );
</script>
<script src="http://www.test.com/friends"></script>
```

这个漏洞在前几年很流行，比如qq邮箱的一个漏洞：http://www.wooyun.org/bugs/wooyun-2010-046

现在的浏览器都已经修复了，可以下载一个Firefox3.0版本来测试下。目前的浏览器在解析JSON Array字符串的时候，不再去触发setter函数了。但对于object.xxx 这样的设置，还是会触发。

## IE的utf-7编码解析问题

这个漏洞也曾经很流行。利用的是老版的IE可以解析utf-7编码的字符串或者文件，绕过服务器的过滤。举个乌云上的例子：http://www.wooyun.org/bugs/wooyun-2011-01293
有这样的一个jsonp调用接口：

http://jipiao.taobao.com/hotel/remote/livesearch.do?callback=%2B%2Fv8%20%2BADwAaAB0AG0APgA8AGIAbwBkAHkAPgA8AHMAYwByAGkAcAB0AD4AYQBsAGUAcgB0ACgAMQApADsAPAAvAHMAYwByAGkAcAB0AD4APAAvAGIAbwBkAHkAPgA8AC8AaAB0AG0APg

url decoder之后是：

http://jipiao.taobao.com/hotel/remote/livesearch.do?callback=+/v8 +ADwAaAB0AG0APgA8AGIAbwBkAHkAPgA8AHMAYwByAGkAcAB0AD4AYQBsAGUAcgB0ACgAMQApADsAPAAvAHMAYwByAGkAcAB0AD4APAAvAGIAbwBkAHkAPgA8AC8AaAB0AG0APg

因为jsonp调用是直接返回callback包装的数据，所以实际上，上面的请求直接返回的是：

```
+/v8 +ADwAaAB0AG0APgA8AGIAbwBkAHkAPgA8AHMAYwByAGkAcAB0AD4AYQBsAGUAcgB0ACgAMQApADsAPAAvAHMAYwByAGkAcAB0AD4APAAvAGIAbwBkAHkAPgA8AC8AaAB0AG0APg-(调用结果数据)
```

IE做了UTF-7解码之后数据是这样子的：

```html
<htm><body><script>alert(1);</script></body></htm>(调用结果数据)
```

于是，就执行了XSS。
另外用IFrame也是可以的。但是我在IE8上测试，url的后缀需要是html才会触发。
IE把没有声明返回Content-Type的请求当做了"text/html"类型的，然后解析就有问题了。

**只要服务器端显式设置了Content-Type为"application/json"，则IE不会识别编码，就不会触发漏洞。所以说服务器端的Content-Type一定要设置对。**尽管设置之后调试有点麻烦，但是却大大提高了安全性。

* JSON格式设置为："application/json"
* JavaScript设置为："application/x-javascript"
* JavaScript还有一些设置为："text/javascript"等，都是不规范的。

## 其它的一些东东

### MongoDB注入

这个实际上就是JSON注入，简单的字符串拼接，可能会引发各种数据被修改的问题。

### JSON解析库的问题

有些JSON库解析库支持循环引用，那么是否可以构造特别的数据，导致其解析失败？从而引起CPU使用过高，拒绝服务等问题？

FastJSON的一个StackOverflowError Bug：

https://github.com/alibaba/fastjson/issues/76

有些JSON库解析有问题：

http://www.freebuf.com/articles/web/10672.html

### JSON-P

有人提出一个JSON-P的规范，但是貌似目前都没有浏览器有支持这个的。
原理是对于JSONP请求，浏览器可以要求服务器返回的MIME是"application/json-p"，这样可以严格校验是否合法的JSON数据。

### CORS（Cross-Origin Resource Sharing）

为了解决跨域调用的安全性问题，目前实际上可用的方案是CORS：

https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS
http://www.w3.org/TR/cors/

原理是通过服务器端设置允许跨域调用，然后浏览器就允许XMLHttpRequest跨域调用了。

CORS可以发起GET/POST请求，不像JSONP，只能发起GET请求。

**默认情况下，CORS请求是不带cookie的。**

我个人认为，这个方案也很蛋疼，一是需要服务器配置，二是协议复杂。浏览器如果不能确定是否能够跨域调用，还要先进行一个Preflight Request。。

**实际上，即使服务器不允许CORS，XMLHttpRequest请求实际上是发送出去，并且返回数据的了，只是浏览器没有让JS环境拿到而已。**

另外，我认为有另外一种数据泄露的可能：黑客可能控制了某个路由，他不能随意抓包，但是他可以在回应头里插入一些特别的头部，比如：

```
Access-Control-Allow-Credentials: true 
```

那么，这时XMLHttpRequest请求就是带cookie的了。

## 最初的问题

回到最初的问题：

* 为什么XMLHttpRequest要遵守同源策略？

    即使XMLHttpRequest是不带Cookie的，也是有可能造成数据泄露的。比如内部网站是根据IP限制访问的，如果XMLHttpRequest不遵守同源策略，那么攻击者可以在用户浏览网页的时候，发起请求，取得内部网站数据。
* XMLHttpRequest 请求会不会带cookie?

    同域情况下会，不同域情况下不会。如果服务器设置Access-Control-Allow-Credentials: true ，也是可以跨域带Cookie的。
* `<script scr="...">` 的标签请求会不会带cookie？

    会。
* 向一个其它域名的网站提交一个form，会不会带cookie？

    会。

## 总结

* 禁止手动拼接JSON字符串，一律应当用JSON库输出。也不应使用自己实现的ObjectToJson等方法，因为可能有各种没有考虑到的地方。
* jsonp请求的callback要严格过滤，只允许"_"，0到9，a-z, A-Z，即合法的javascript函数的命名。
* jsonp请求也要判断合法性，比如用户是否登陆（这点很容易被忽略）。
* 设置好Content-Type（这点对于调试不方便，但是提高了安全性）。
* 以jsonp方式调用第三方的接口，实际相当于引入了第三方的JS代码，要慎重。

## 参考

* http://www.json.org/
* http://www.slideshare.net/wurbanski/nosql-no-security
* https://github.com/douglascrockford/JSON-js
* http://toolswebtop.com/      在线编码转换，可以转换UTF-7    
* http://www.thespanner.co.uk/2011/05/30/json-hijacking/
* http://www.thespanner.co.uk/2009/11/23/bypassing-csp-for-fun-no-profit/
* http://stackoverflow.com/questions/1830050/why-same-origin-policy-for-xmlhttprequest
