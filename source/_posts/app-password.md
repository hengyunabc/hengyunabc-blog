---
title: 移动App该如何保存用户密码
date: 2012-07-19 19:58:28
tags:
 - app
 - security
 - password


categories:
  - 技术

---

## 更新

update 2018-06-04

2015年出的一个规范 JSON Web Token (JWT)  https://tools.ietf.org/html/rfc7519 

JWT 官网： https://jwt.io/  

八幅漫画理解使用JSON Web Token设计单点登录系统： http://blog.leapoahead.com/2015/09/07/user-authentication-with-jwt/ 

JSON Web Encryption (JWE) ： https://tools.ietf.org/html/rfc7516 

JSON Web Signature (JWS) ： https://tools.ietf.org/html/rfc7515 



update 2017-9-6 

微信交互协议和加密模式研究：https://github.com/hengyunabc/hengyunabc.github.io/files/1280081/wechat.pdf

## 背景

移动App该如何保存用户密码？

这个实际上和桌面程序是一样的。

## 先看下一些软件是如何保存用户密码的

* QQ

参考：http://bbs.pediy.com/archive/index.php?t-159045.html，桌面QQ在2012的时候把密码md5计算之后，保存到本地加密的Sqlite数据库里。

* 手机淘宝

参考：http://blog.csdn.net/androidsecurity/article/details/8666954

手机淘宝是通过本地DES加密，再把密码保存到本地文件里的，如果拿到ROOT权限，能破解出密码明文。

* Windows

参考：http://www.freebuf.com/tools/37162.html 

我实际测试了下，可以轻松得到所有帐号的密码明文。

* Linux

参考：http://blog.csdn.net/lqhbupt/article/details/7787802

**linux是通过加盐(salt)，再hash后，保存到/etc/shadow文件里的。**

貌似以前的发行版是md5 hash，现在的发行版都是SHA-512 hash。

linux用户密码的hash算法： http://serverfault.com/questions/439650/how-are-the-hashes-in-etc-shadow-generated

实际上是调用了glic里的crypt函数，可以在man手册里查看相关的信息。

可以用下面的命令来生成：

```
mkpasswd --method=SHA-512 --salt=xxxx
```

其中salt参数，可以自己设置，最好是随机生成的。
可以用 `mkpasswd --method=help` 来查看支持的算法。

## 用户密码该如何保存，还有能做到哪种程度？

看完上面一些软件的做法之后，我们来探讨下，用户密码该如何保存，还有能做到哪种程度？

* 假定本地存储的hash串/加密串，和加密算法，攻击者都可以得到，或者逆向分析到。

实际上也是如此，通过上面QQ和淘宝的例子，允分说明了加密串是可以得到的。Linux更是一切都是公开的，只要有权限就可以读取到，包括salt值，shah算法，(salt+密码) hash之后的结果。

* 防止攻击者得到用户密码的明文。这个实际上是从用户的角度出发，即使数据泄露了，影响降到最低。
* 防止攻击者拿到hash串或者加密串之后，一直都可以登陆。这点对于移动设置是很重要的，比如今天用户连到了一个恶意的wifi，如果攻击者截获到请求，要防止攻击者潜伏几天，或者几个月之后的攻击。必须要让请求的凭据在一天或者几天内失效。

## 加盐(salt)

假如不加盐，那么攻击者可以根据同样的hash值得到很多信息。

比如网站1的数据库泄露了，攻击者发现用户A和用户B的hash值是一样的，然后攻击者通过其它途径拿到了用户A的密码，那么攻击者就可以知道用户B的密码了。

或者攻击者通过彩虹表，暴力破解等方式可以直接知道用户的原来密码。

**所以，每个用户的salt值都要是不一样的，这点参考linux的/etc/shadow文件就知道了。**

## 客户端本地存储密码的算法

应该用哪种算法来存储？

从上面的资料来看，手机淘宝是本地DES对称加密，显然很容易就可以破解到用户的真实密码。QQ也是对称加密的数据库里，存储了用户密码的md5值。

显然对称加密算法都是可以逆向得到原来的数据的。那么我们尝试用非对称加密算法，比如RSA来传输用户的密码。

那么用户登陆的流程就变为：

1. 客户端用公钥加密用户密码，保存到本地；
1. 用户要登陆时，发送加密串到服务器；
1. 服务器用私钥解密，得到用户的密码，再验证。

有的人会说，如果服务器的私钥泄露怎么办？

服务器端换个新的密钥，强制客户端下载新的公钥或者升级。

**可以考虑有一个专门的硬件来解密，这个硬件只负责计算，私钥是一次性写入不可读取和修改的。搜索 rsa hardware，貌似的确有这样的硬件。**

当然，即使真的私钥泄露，世界一样运转，像OpenSSL的心血漏洞就可能泄露服务器私钥，但大家日子一样过。

非对称加密算法的好处：

* 即使数据被盗，攻击者拿不到密码的明文
* 如果发现有部分用户的数据被盗了（公钥加密后的数据），可以通过升级服务器和客户端的版本，让用户重新输入密码，用户还是原来的密码，但是攻击者却登陆不了了。
* 对于安全要求严格的应用，还可以定期更新私钥，来保证用户的数据安全。

## 如何防止本地加密串泄露之后，攻击者可能潜伏很久？

这点实际上是如何让客户端保存的加密串及时的失效。

比如：

1. 强制要求客户端保存的加密串一周失效；
1. 用户手机中病毒了，攻击者窃取到了加密串。但是清除病毒之后，用户没有够时的修改密码。攻击者是否会潜伏很久？
1. **发现某木马大规模窃取到了大量的用户本地加密串，是否可以强制用户的本地加密串失效，客户端不用升级，用户不用修改密码，也不会泄露信息？**

下面提出一种 salt + 非对称加密算法的方案来解决这个问题：

1. 用户填写密码，客户端随机生成一个salt值（注意这个salt只是防止中间人拦截到原始的password的加密串），用公钥把 (salt + password)加密，设置首次登陆的参数，发送到服务器；
1. 服务器检查参数，发现是首次登陆，则服务器用私钥解密，得到password（抛弃salt值），验证，如果通过，则随机生成一个salt值，并把salt值保存起来（保存到缓存里，设置7天过期），然后用公钥把(salt + 用户名)加密，返回给客户端。
1. 客户端保存服务器返回的加密串，完成登陆。
1. 客户端下次自动登陆时，把上次保存的加密串直接发给服务器，并设置二次登陆的参数。
1. 服务器检查参数，发现是二次登陆，用私钥解密，得到salt + 用户名，然后检查salt值是否过期了（到缓存中查找，如果没有，即过期），如果过期，则通知客户端，让用户重新输入密码。如果没有过期，再验证密码是否正确。如果正确，则通知客户端登陆成功。
1. **如果发现某帐户异常，可以直接清除缓存中对应用户的salt值，这样用户再登陆就会失败。同理，如果某木马大规模窃取到了大量的用户本地加密串，那么可以把缓存中所有用户的salt都清除，那么所有用户都要重新登陆。注意用户的密码不用修改。**
1. 第2步中服务器生成的salt值，可以带上用户的mac值，os版本等，这样可以增强检验。

注意，为了简化描述，上面提到的用户的password，可以是先用某个hash算法hash一次。

## 具体的登陆流程

### 浏览器登陆的流程

浏览器的登陆过程比较简单，只要用RSA公钥加密密码就可以了。防止中间人截取到明文的密码。 

![](/img/browser-login.png)

### App登陆保存数据流程

App因为要实现自动登陆功能，所以必然要保存一些凭据，所以比较复杂。 

App登陆要实现的功能： 

* 密码不会明文存储，并且不能反编绎解密； 
* 在服务器端可以控制App端的登陆有效性，防止攻击者拿到数据之后，可以长久地登陆； 
* 用户如果密码没有泄露，不用修改密码就可以保证安全性； 
* 可以区分不同类型的客户端安全性；比如Android用户受到攻击，只会让Android用户的登陆失效，IOS用户不受影响。 

### App第一次登陆流程

* 用户输入密码，App把这些信息用RSA公钥加密：(用户名,密码,时间,mac,随机数)，并发送到服务器。 
* 服务器用RSA私钥解密，判断时间（可以动态调整1天到7天），如果不在时间范围之内，则登陆失败。如果在时间范围之内，再调用coreservice判断用户名和密码。

这里判断时间，主要是防止攻击者截取到加密串后，可以长久地利用这个加密串来登陆。 

* 如果服务器判断用户成功登陆，则用AES加密：(随机salt,用户名,客户端类型,时间)，以（用户名+Android/IOS/WP）为key，存到缓存里。再把加密结果返回给客户端。 
* 客户端保存服务器返回的加密串 

### App自动登陆的流程

* App发送保存的加密串到服务器，（加密串，用户名，mac，随机数）==>RSA公钥加密 
* 服务器用RSA私钥解密，再用AES解密加密串，判断用户名是否一致。如果一致，再以（用户名+Android/IOS/WP）为key到缓存里查询。如果判断缓存中的salt值和客户端发送过来的一致，则用户登陆成功。否则登陆失败。 

不用AES加密，用RSA公钥加密也是可以的。AES速度比RSA要快，RSA只能存储有限的数据。

![](/img/app-login.png)

## 其它的一些东东

多次md5或者md5 + sha1是没什么效果的。

RSA算法最好选择2048位的。搜索" rsa 1024 crack"有很多相关的结果，google已经将其SSL用的RSA算法升级为2048位的。

如何防止登陆过程的中间人攻击，可以参考，魔兽世界的叫SPR6的登陆算法。

## 总结

对于网页登陆，可以考虑支持多种方式：

* 不支持JS的，用原始密码登陆。
* 支持JS的，可以考虑传递hash算法加密字符串。严格要求的应用，最好用JS实现RSA加密。在github上找到的一个JS RSA库：https://github.com/travist/jsencrypt
* 客户端应用，一律应当用RSA算法，并加盐来保存用户密码。单纯的hash或者对称加密算法都不靠谱。

服务器用salt（存数据库的） + hash算法来保存用户的密码。

用salt（存缓存的，注意和上一行的salt是不同的）+ RSA算法来加密用户登陆的凭证。

这样服务器可以灵活控制风险，控制用户登陆凭据的有效期，即使用户数据泄露，也不需要修改密码。
