layout: post
title: 科学上网的一些原理
date: 2015-02-08 10:26:37
tags:
- proxy
- 代理

categories: 
 - 网络

---

##知其所以然

本文不是教程向，倾向于分析科学上网的一些原理。知其所以然，才能更好地使用工具，也可以创作出自己的工具。

科学上网的工具很多，八仙过海，各显神通，而且综合了各种技术。尝试从以下四个方面来解析一些其中的原理。大致先原理，再工具的顺序。

- dns
- http/https proxy
- vpn
- socks proxy


##一个http请求发生了什么？
这个是个比较流行的面试题，从中可以引出很多的内容。大致分为下面四个步骤：
- dns解析，得到IP
- 向目标IP发起TCP请求
- 发送http request
- 服务器回应，浏览器解析

还有很多细节，更多参考：

http://fex.baidu.com/blog/2014/05/what-happen/

http://stackoverflow.com/questions/2092527/what-happens-when-you-type-in-a-url-in-browser

http://div.io/topic/609?page=1  从FE的角度上再看输入url后都发生了什么

##DNS/域名解析
可以看到dns解析是最初的一步，也是最重要的一步。比如访问亲友，要知道他的正确的住址，才能正确地上门拜访。

dns有两种协议，一种是UDP（默认），一种是TCP。

###udp 方式，先回应的数据包被当做有效数据
在linux下可以用dig来检测dns。国内的DNS服务器通常不会返回正常的结果。
下面以google的8.8.8.8 dns服务器来做测试，并用wireshark来抓包，分析结果。
```
dig @8.8.8.8  www.youtube.com
```
![dns-udp-youtube](/img/dns-udp-youtube-badresponse-fast.png)

从wireshark的结果，可以看到返回了三个结果，**前面两个是错误的，后面的是正确的**。

但是，对于dns客户端来说，它只会取最快回应的的结果，后面的正确结果被丢弃掉了。**因为中间被插入了污染包，所以即使我们配置了正确的dns服务器，也解析不到正确的IP。**

###tcp 方式，有时有效，可能被rest
再用TCP下的DNS来测试下:
```
dig @8.8.8.8 +tcp   www.youtube.com
```
![dns-tcp-youtube-reset](/img/dns-tcp-youtube-reset.png)

从wireshark的结果，可以看出在TCP三次握手成功时，本地发出了一个查询www.youtube.com的dns请求，结果，很快收到了一个RST回应。而RST回应是在TCP连接断开时，才会发出的。所以可以看出，**TCP通讯受到了干扰，DNS客户端因为收到RST回应，认为对方断开了连接，因此也无法收到后面正确的回应数据包了。**

再来看下解析twitter的结果：
```
dig @8.8.8.8 +tcp  www.twitter.com
```
结果：
```
www.twitter.com.        590     IN      CNAME   twitter.com.
twitter.com.            20      IN      A       199.59.150.7 80
twitter.com.            20      IN      A       199.59.150.7
twitter.com.            20      IN      A       199.59.149.230
twitter.com.            20      IN      A       199.59.150.39

```
这次返回的IP是正确的。但是尝试用telnet 去连接时，会发现连接不上。
```
telnet 199.59.150.7 80
```
但是，在国外服务器去连接时，可以正常连接，完成一个http请求。**可见一些IP的访问被禁止了**。
```
$ telnet 199.59.150.7 80
Trying 199.59.150.7...
Connected to 199.59.150.7.
Escape character is '^]'.
GET / HTTP/1.0
HOST:www.twitter.com

HTTP/1.0 301 Moved Permanently
content-length: 0
date: Sun, 08 Feb 2015 06:28:08 UTC
location: https://www.twitter.com/
server: tsa_a
set-cookie: guest_id=v1%3A142337688883648506; Domain=.twitter.com; Path=/; Expires=Tue, 07-Feb-2017 06:28:08 UTC
x-connection-hash: 0f5eab0ea2d6309109f15447e1da6b13
x-response-time: 2
```

###黑名单/白名单
想要获取到正确的IP，自然的黑名单/白名单两种思路。

下面列出一些相关的项目：
```
https://github.com/holmium/dnsforwarder
https://code.google.com/p/huhamhire-hosts/
https://github.com/felixonmars/dnsmasq-china-list
```
###本地DNS软件
- 修改hosts文件
相信大家都很熟悉，也有一些工具可以自动更新hosts文件的。
- 浏览器pac文件
主流浏览器或者其插件，都可以配置pac文件。pac文件实际上是一个JS文件，可以通过编程的方式来控制dns解析结果。其效果类似hosts文件，不过pac文件通常都是由插件控制自动更新的。只能控制浏览器的dns解析。
- 本地dns服务器，dnsmasq
在linux下，可以自己配置一个dnsmasq服务器，然后自己管理dns。不过比较高级，也比较麻烦。

顺便提一下，实际上，kubuntu的NetworkManager会自己启动一个私有的dnsmasq进程来做dns解析。不过它侦听的是127.0.1.1，所以并不会造成冲突。
```
/usr/sbin/dnsmasq --no-resolv --keep-in-foreground --no-hosts --bind-interfaces --pid-file=/run/sendsigs.omit.d/network-manager.dnsmasq.pid --listen-address=127.0.1.1 --conf-file=/var/run/NetworkManager/dnsmasq.conf
```
###路由器智能DNS
基于OpenWRT/Tomoto的路由器可以在上面配置dns server，从而实现在路由器级别智能dns解析。现在国内的一些路由器是基于OpenWRT的，因此支持配置dns服务器。
参考项目：
```
https://github.com/clowwindy/ChinaDNS
```

## http proxy

### http proxy请求和没有proxy的请求的区别
在chrome里没有设置http proxy的请求头信息是这样的：
```
GET /nocache/fesplg/s.gif
Host: www.baidu.com
```
在设置了http proxy之后，发送的请求头是这样的：
```
GET http://www.baidu.com//nocache/fesplg/s.gif
Host: www.baidu.com
Proxy-Connection: keep-alive
```
区别是配置http proxy之后，会在请求里发送完整的url。

client在发送请求时，如果没有proxy，则直接发送path，如果有proxy，则要发送完整的url。

实际上http proxy server可以处理两种情况，即使客户端没有发送完整的url，因为host字段里，已经有host信息了。

为什么请求里要有完整的url？

历史原因。

### 目标服务器能否感知到http proxy的存在？
当我们使用http proxy时，有个问题可能会关心的：目标服务器能否感知到http proxy的存在？

一个配置了proxy的浏览器请求头：
```
GET http://55.75.138.79:9999/ HTTP/1.1
Host: 55.75.138.79:9999
Proxy-Connection: keep-alive
```
实际上目标服务器接收到的信息是这样子的：
```
GET / HTTP/1.1
Host: 55.75.138.79:9999
Connection: keep-alive
```
可见，http proxy服务器并没有把proxy相关信息发送到目标服务器上。

因此，目标服务器是没有办法知道用户是否使用了http proxy。

### http proxy keep-alive
实际上Proxy-Connection: keep-alive这个请求头是错误的，不在标准里：

因为http1.1 默认就是Connection: keep-alive

如果client想要http proxy在请求之后关闭connection，可以用Proxy-Connection: close 来指明。

http://homepage.ntlworld.com/jonathan.deboynepollard/FGA/web-proxy-connection-header.html

###  http proxy authentication
当http proxy需要密码时：

第一次请求没有密码，则会回应
```
HTTP/1.1 407 Proxy authentication required
Proxy-Authenticate: Basic realm="Polipo"
```
浏览器会弹出窗口，要求输入密码。
如果密码错误的话，回应头是：
```
HTTP/1.1 407 Proxy authentication incorrect
```
如果是配置了密码，发送的请求头则是：
```
GET http://www.baidu.com/ HTTP/1.1
Host: www.baidu.com
Proxy-Connection: keep-alive
Proxy-Authorization: Basic YWRtaW46YWRtaW4=
```
Proxy-Authorization实际是Base64编码。
```
base64("admin:admin") == "YWRtaW46YWRtaW4="
```
### http proxy对于不认识的header和方法的处理：
http proxy通常会尽量原样发送，因为很多程序都扩展了http method，如果不支持，很多程序都不能正常工作。

客户端用OPTIONS 请求可以探测服务器支持的方法。但是意义不大。

##https proxy
当访问一个https网站时，https://github.com

先发送connect method，如果支持，会返回200

```
CONNECT github.com:443 HTTP/1.1
Host: github.com
Proxy-Connection: keep-alive
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/39.0.2171.95 Safari/537.36
 
HTTP/1.1 200 OK
```
###http tunnel
http://en.wikipedia.org/wiki/HTTP_tunnel#HTTP_CONNECT_tunneling 

通过connect method，http proxy server实际上充当tcp转发的中间人。
比如，用nc 通过http proxy来连42端口：
```bash
$ nc -x10.2.3.4:8080 -Xconnect host.example.com 42 
```

原理是利用CONNECT方法，让http proxy服务器充当中间人。

###https proxy的安全性？
proxy server可以拿到什么信息？

通过一个http proxy去访问支付宝是否安全？

- 可以知道host，即要访问的是哪个网站
- 拿不到url信息
- https协议保证不会泄露通信内容
- TLS(Transport Layer Security)  在握手时，生成强度足够的随机数
- TLS 每一个record都要有一个sequence number，每发一个增加一个，并且是不能翻转的。
- TLS 保证不会出现重放攻击

TLS的内容很多，这里说到关于安全的一些关键点。

注意事项：
- 确保是https访问
- 确保访问网站的证书没有问题

是否真的安全了？更强的攻击者！

流量劫持 —— 浮层登录框的隐患

http://fex.baidu.com/blog/2014/06/danger-behind-popup-login-dialog/

**所以，尽量不要使用来路不明的http/https proxy，使用公开的wifi也要小心。**


##goagent工作原理

- local http/https proxy
- 伪造https证书，导入浏览器信任列表里
- 浏览器配置http/https proxy
- 解析出http/https request的内容。然后把这些请求内容打包，发给GAE服务器
- 与GAE通信通过http/https，内容用RC4算法加密
- GAE服务器，再调用google提供的 urlfetch，来获得请求的回应，然后再把回应打包，返回给客户端。
- 客户端把回应传给浏览器
- 自带dns解析服务器
- 在local/certs/ 目录下可以找到缓存的伪造的证书

fiddler抓取https数据包是同样原理。

goagent会为每一个https网站伪造一个证书，并缓存起来。比如下面这个github的证书：
![goagent-github-cert.png](/img/goagent-github-cert.png)

goagent的代码在3.0之后，支持了很多其它功能，变得有点混乱了。

以3.2.0 版本为例：

主要的代码是在server/gae/gae.py 里。
https://github.com/goagent/goagent/blob/v3.2.0/server/gae/gae.py#L107

一些代码实现的细节：
- 支持最长的url是2083，因为gae sdk的限制。
https://github.com/AppScale/gae_sdk/blob/master/google/appengine/api/taskqueue/taskqueue.py#L241
- 如果回应的内容是/text, json, javascript，且 > 512会用gzip来压缩
- 处理一些Content-Range 的回应内容。Content-Range 的代码虽然只有一点点，但是如果是不熟悉的话，要花上不少工夫。
- goagent的生成证书的代码在 local/proxylib.py的这个函数里：

```python
    @staticmethod
    def _get_cert(commonname, sans=()):
```

###为什么goagent可以看视频？

因为很多网站都是http协议的。有少部分是rmtp协议的，也有是rmtp over http的。

在youku看视频的一个请求数据：
```
http://14.152.72.22/youku/65748B784A636820C5A81B41C7/030002090454919F64A167032DBBC7EE242548-46C9-EB9D-916D-D8BA8D5159D3.flv?&start=158
response：
Connection:close
Content-Length:7883513
Content-Type:video/x-flv
Date:Wed, 17 Dec 2014 17:55:24 GMT
ETag:"316284225"
Last-Modified:Wed, 17 Dec 2014 15:21:26 GMT
Server:YOUKU.GZ
```
可以看到，有ETag，有长度信息等。

###goagent缺点
- 只是http proxy，不能代理其它协议
- google的IP经常失效
- 不支持websocket协议
- 配置复杂

## vpn

###流行的vpn类型
- PPTP，linux pptpd，安全性低，不能保证数据完整性或者来源，MPPE加密暴力破解
- L2TP，linux xl2tpd，预共享密钥可以保证安全性
- SSTP，基于HTTPS，微软提出。linux开源实现SoftEther VPN
- OPENVPN，基于SSL，预共享密钥可以保证安全性
- 所谓的SSL VPN，各家厂商有自己的实现，没有统一的标准
- 新型的staless VPN，像sigmavpn/ShadowVPN等

现状：
- PPTP/L2TP 可用，但可能会不管用
- SoftEther VPN/OPENVPN 可能会导致服务器被封IP，连不上，慎用
- ShadowVPN可用，sigmavpn没有测试

猜测下为什么PPTP，L2TP这些方案容易被检测到？

可能是因为它们的协议都有明显的标头：
- 转发的是ppp协议数据，握手有特征
- PPTP协议有GRE标头和PPP标头
- L2TP有L2TP标头和PPP标头
- L2TP要用到IPsec

参考：

https://technet.microsoft.com/zh-cn/library/cc771298(v=ws.10).aspx

###网页版的SSL VPN
有些企业，或者学校里，会有这种VPN：
 - 网页登陆帐号
 - 设置IE代理，为远程服务器地址
 - 通过代理浏览内部网页
 
这种SSL VPN原理很简单，就是一个登陆验证的http proxy，其实并不能算是VPN？

###新型的staless vpnVPN，sigmavpn/ShadowVPN

这种新型VPN的原理是，利用虚拟的网络设备TUN和TAP，把请求数据先发给虚拟设备，然后把数据加密转发到远程服务器。（VPN都这原理？）

```
you <-> local <-> protocol <-> remote <-> ...... <-> remote <-> protocol <-> local <-> peer
```

这种新型VPN的特点是很轻量，没有传统VPN那么复杂的握手加密控制等，而向个人，而非企业。SigmaVPN号称只有几百行代码。

参考：

http://zh.wikipedia.org/wiki/TUN%E4%B8%8ETAP

https://code.google.com/p/sigmavpn/wiki/Introduction

### ubuntu pptp vpn server安装
ubuntu官方参考文档：
https://help.ubuntu.com/community/PPTPServer 

- vps 要开启ppp和nat网络转发的功能

- **设置MTU，建议设置为1200以下**，因为中间网络可能很复杂，MTU太大可能导致连接失败
```
iptables -A FORWARD -p tcp --syn -s 192.168.0.0/24 -j TCPMSS --set-mss 1200
```

## socks proxy
- rfc 文档： http://tools.ietf.org/html/rfc1928
- wiki上的简介： http://en.wikipedia.org/wiki/SOCKS#SOCKS5

- socks4/socks4a 已经过时
- socks5

socks5支持udp，所以如果客户端把dns查询也走socks的话，那么就可以直接解决dns的问题了。
###socks proxy 握手的过程
socks5流程
- 客户端查询服务器支持的认证方式
- 服务器回应支持的认证方式
- 客户端发送认证信息，服务器回应
- 如果通过，客户端直接发送TCP/UDP的原始数据，以后proxy只单纯转发数据流，不做任何处理了

- **socks proxy 自身没有加密机制，简单的TCP/UDP forward**

socks协议其实是相当简单的，用wireshark抓包，结合netty-codec-socks，很容易可以理解其工作过程。
https://github.com/netty/netty/tree/master/codec-socks

###ssh socks proxy
如果有一个外国的服务器，可以通过ssh连接登陆，那么可以很简单地搭建一个本地的socks5代理。

XShell可以通过“转移规则”来配置本地socks服务器，putty也有类似的配置：
![xshell-sock5-proxy.png](/img/xshell-sock5-proxy.png)

linux下命令行启动一个本地sock5服务器：
```bash
ssh -D 1080 user@romoteHost
```
ssh还有一些端口转发的技巧，这对于测试网络程序，绕过防火墙也是很有帮助的。

参考：http://www.ibm.com/developerworks/cn/linux/l-cn-sshforward/

###shadowsocks的工作原理
shadowsocks是非常流行的一个代理工具，其原理非常简单。

- 客户端服务器预共享密码
- 本地socks5 proxy server（有没有想起在学校时用的ccproxy？）
- 软件/浏览器配置本地socks代理
- 本地socks server把数据包装，AES256加密，发送到远程服务器
- 远程服务器解密，转发给对应的服务器

```
app => local socks server(encrypt) => shadowsocks server(decrypt) => real host
                                                                       
app <= (decrypt) local socks server <= (encrypt) shadowsocks server <= real host
```

其它的一些东东：
- 一个端口一个密码，没有用户的概念
- 支持多个workder并发
- 协议简单，比socks协议还要简单，抽取了socks协议的部分

###shadowsoks的优点
- 中间没有任何握手的环节，直接是TCP数据流
- 速度快


###shadowsocks的安全性
- 服务器可以解出所有的TCP/UDP数据
- 中间人攻击，重放攻击

所以，**对于第三方shadow socks服务器，要慎重使用。**

在使用shadowsocks的情况下，https通迅是安全的，但是仍然有危险，参见上面http proxy安全的内容。

###vpn和socks代理的区别
从原理上来说，socks代理会更快，因为转发的数据更少。

因为vpn转发的是ppp数据包，ppp协议是数据链路层(data link layer)的协议。socks转发的是TCP/UDP数据，是传输(transport)层。

VPN的优点是很容易配置成全局的，这对于很多不能配置代理的程序来说很方便。而配置全局的socks proxy比较麻烦，目前貌似还没有简单的方案。

###linux下一些软件配置代理的方法

- bash/shell

对于shell，最简单的办法是在命令的前面设置下http_porxy的环境变量。
```bash
http_proxy=http://127.0.0.1:8123 wget http://test.com
```
推荐的做法是在~/.bashrc 文件里设置两个命令，开关http proxy：
```bash
alias proxyOn='export https_proxy=http://127.0.0.1:8123 && http_proxy=http://127.0.0.1:8123'
alias proxyOff='unset https_proxy && unset  http_proxy'
```
注意，如果想sudo的情况下，http proxy仍然有效，要配置env_keep。

在/etc/sudoers.d/目录下增加一个env_keep的文件，内容是：
```
Defaults env_keep += " http_proxy https_proxy ftp_proxy "
```

参考：
https://help.ubuntu.com/community/AptGet/Howto#Setting_up_apt-get_to_use_a_http-proxy

- GUI软件

现在大部分软件都可以设置代理。
gnome和kde都可以设置全局的代理。

###linux下不支持代理的程序使用socks代理：tsocks
tsocks利用LD_PRELOAD机制，代理程序里的connect函数，然后就可以代理所有的TCP请求了。
不过dns请求，默认是通过udp来发送的，所以tsocks不能代理dns请求。

默认情况下，tsocks会先加载~/.tsocks.conf，如果没有，再加载/etc/tsocks.conf。对于local ip不会代理。

使用方法：

```bash
sudo apt-get install tsocks
LD_PRELOAD=/usr/lib/libtsocks.so wget http://www.facebook.com
```



##基于路由器的方案

基于路由器的方案有很多，原理和本机的方案是一样的，只不过把这些措施前移到路由器里。

路由器的方案的优点是很明显的：
- 手机/平板不用设置
- 公司/局域网级的代理

但是需要专门的路由器，刷固件等。

shadowsocks, shadowvpn都可以跑在路由器上。

一些项目收集：

https://github.com/lifetyper/FreeRouter_V2

https://gist.github.com/wen-long/8644243

https://github.com/ashi009/bestroutetb 


## 推荐的办法
完全免费
- chrome + switchsharp + http proxy
- goagent

程序员的推荐
- chrome + switchsharp + socks5 proxy
- aws免费一年的服务器/其它国外免费云主机，节点位置决定速度，推荐东京机房
- shadowsocks

第三方免费的服务器
- shadowsocks服务器，微信公众号：pennyjob

手机软件：
- fqrouter
- shadowsocks client

商业软件安全性自己考虑


##总结

- 新技术层出不穷
- 越流行，越容易失效
- 实现一个proxy其实相当简单
- 知其所以然，更好使用工具，也可以创作出自己的工具。

