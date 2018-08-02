title: 防止页面被iframe恶意嵌套
date: 2015-03-04 16:19:19
tags:
 - iframe
 - html
 - javascript
 - 安全
  
categories:
 - 技术
 
---

## 缘起

在看资料时，看到这样的防止iframe嵌套的代码：
```javascript
try {
    if (window.top != window.self) {
        var ref = document.referer;
        if (ref.substring(0, 2) === '//') {
            ref = 'http:' + ref;
        } else if (ref.split('://').length === 1) {
            ref = 'http://' + ref;
        }
        var url = ref.split('/');
        var _l = {auth: ''};
        var host = url[2].split('@');
        if (host.length === 1) {
            host = host[0].split(':');
        } else {
            _l.auth = host[0];
            host = host[1].split(':');
        }
        var parentHostName = host[0];
        if (parentHostName.indexOf("test.com") == -1 && parentHostName.indexOf("test2.com") == -1) {
            top.location.href = "http://www.test.com";
        }
    }
} catch (e) {
}
```
假定test.com，test2.com是自己的域名，当其它网站恶意嵌套本站的页面时，跳转回本站的首页。

上面的代码有两个问题：

- referer拼写错误，实际上应该是referrer
- 解析referrer的代码太复杂，还不一定正确

**无论在任何语言里，都不建议手工写代码处理URL。因为url的复杂度超出一般人的想像。很多安全的问题就是因为解析URL不当引起的。比如防止CSRF时判断referrer。**

URI的语法：

http://en.wikipedia.org/wiki/URI_scheme#Generic_syntax

## 在javascript里解析url最好的办法

在javascript里解析url的最好办法是利用浏览器的js引擎，通过创建一个a标签：
```javascript
var getLocation = function(href) {
    var l = document.createElement("a");
    l.href = href;
    return l;
};
var l = getLocation("http://example.com/path");
console.debug(l.hostname)
```

## 简洁防iframe恶意嵌套的方法

下面给出一个简洁的防止iframe恶意嵌套的判断方法：
```javascript
if(window.top != window && document.referrer){
  var a = document.createElement("a");
  a.href = document.referrer;
  var host = a.hostname;

  var endsWith = function (str, suffix) {
      return str.indexOf(suffix, str.length - suffix.length) !== -1;
  }

  if(!endsWith(host, '.test.com') || !endsWith(host, '.test2.com')){
    top.location.href = "http://www.test.com";
  }
}
```

## java里处理URL的方法
http://docs.oracle.com/javase/tutorial/networking/urls/urlInfo.html

用contain, indexOf, endWitch这些函数时都要小心。

```java
 public static void main(String[] args) throws Exception {

        URL aURL = new URL("http://example.com:80/docs/books/tutorial"
                           + "/index.html?name=networking#DOWNLOADING");

        System.out.println("protocol = " + aURL.getProtocol());
        System.out.println("authority = " + aURL.getAuthority());
        System.out.println("host = " + aURL.getHost());
        System.out.println("port = " + aURL.getPort());
        System.out.println("path = " + aURL.getPath());
        System.out.println("query = " + aURL.getQuery());
        System.out.println("filename = " + aURL.getFile());
        System.out.println("ref = " + aURL.getRef());
    }
```

## 参考

http://stackoverflow.com/questions/736513/how-do-i-parse-a-url-into-hostname-and-path-in-javascript

http://stackoverflow.com/questions/5522097/prevent-iframe-stealing
