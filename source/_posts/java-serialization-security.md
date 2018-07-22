---
title: 禁止JVM执行外部命令Runtime.exec -- 由Apache Commons Collections漏洞引发的思考
date: 2015-11-13 16:58:28
tags:
 - java
 - jvm
 - security


categories:
  - 技术

---


update: 2015-11-16
## 新版apache commons collections 3.2.2修复漏洞

新版本的apache commons collections默认禁止了不安全的一些转换类。可以通过升级来修复漏洞。参考release说明：https://commons.apache.org/proper/commons-collections/release_3_2_2.html


## Dubbo rpc远程代码执行的例子
update: 2015-11-13

重新思考了下这个漏洞，给出一个dubbo rpc远程代码执行的例子。

https://github.com/hengyunabc/dubbo-apache-commons-collections-bug

**可以说很多公司开放的rpc，只要协议里支持java序列化方式，classpath里有apache commons collections的jar包，都存在被远程代码执行的风险。**

至于能不能通过http接口再调用dubbo rpc远程代码，貌似不太可行。因为信息难以传递。

----

## Apache Commons Collections远程代码执行漏洞

最近出来一个比较严重的漏洞，在使用了Apache Commons Collections的Java应用，可以远程代码执行。包括最新版的WebLogic、WebSphere、JBoss、Jenkins、OpenNMS这些大名鼎鼎的Java应用。

这个漏洞的严重的地方在于，即使你的代码里没有使用到Apache Commons Collections里的类，只要Java应用的Classpath里有Apache Commons Collections的jar包，都可以远程代码执行。

参考：

* https://github.com/frohoff/ysoserial
* http://blog.chaitin.com/2015-11-11_java_unserialize_rce/

这个漏洞的演示很简单，只要在maven依赖里增加
```xml
		<dependency>
			<groupId>commons-collections</groupId>
			<artifactId>commons-collections</artifactId>
			<version>3.2.1</version>
		</dependency>
```
再执行下面的java代码：
```java
        Transformer[] transformers = new Transformer[] { new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] { String.class, Class[].class },
                        new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class, Object[].class },
                        new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class }, new Object[] { "calc" }) };

        Transformer transformedChain = new ChainedTransformer(transformers);

        Map innerMap = new HashMap();
        innerMap.put("value", "value");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformedChain);

        Map.Entry onlyElement = (Entry) outerMap.entrySet().iterator().next();
        onlyElement.setValue("foobar");
```

这个漏洞的根本问题并不是Java序列化的问题，而是Apache Commons Collections允许链式的任意的类函数反射调用。攻击者通过允许Java序列化协议的端口，把攻击代码上传到服务器上，再由Apache Commons Collections里的TransformedMap来执行。

这里不对这个漏洞多做展开，可以看上面的参考文章。

## 如何简单的防止Java程序调用外部命令？

从这个漏洞，引发了很久之前的一个念头：**如何简单的防止Java程序调用外部命令？**

java相对来说安全性问题比较少。出现的一些问题大部分是利用反射，最终用Runtime.exec(String cmd)函数来执行外部命令的。**如果可以禁止JVM执行外部命令，未知漏洞的危害性会大大降低，可以大大提高JVM的安全性。**

换而言之，就是如何禁止Java执行Runtime.exec(String cmd)之类的函数。

在Java里有一套Security Policy，但是实际上用的人比较少。因为配置起来太麻烦了。
参考：

* http://docs.oracle.com/javase/8/docs/technotes/guides/security/PolicyFiles.html
* http://docs.oracle.com/javase/8/docs/technotes/guides/security/permissions.html
* http://docs.gigaspaces.com/xap102sec/java-security-policy-file.html 详细的权限列表可以参考这个文档

从文档里可以知道，Java里并没有直接禁止Rumtine.exec 函数执行的权限。

禁止文件执行的权限在java.io.FilePermission里。如果想禁止所有外部文件执行，可以在下面的配置文件中把````execute```` 删除：
```
grant {
  permission java.io.FilePermission "<<ALL FILES>>", "read, write, delete, execute";
};
```
但是**Java权限机制是白名单的**，还有一大堆的权限要配置上去，非常复杂。
从tomcat的配置就知道了。http://tomcat.apache.org/tomcat-7.0-doc/security-manager-howto.html
所以Tomcat默认是没有启用Security Policy的，可以通过在命令加上-security参数来启用。
```bash
catalina.sh start -security
```

那么有没有简单的办法可以在代码里禁止Java执行外部命令？

研究了下，通过扩展SecurityManager可以简单实现：

```java
        SecurityManager originalSecurityManager = System.getSecurityManager();
        if (originalSecurityManager == null) {
            // 创建自己的SecurityManager
            SecurityManager sm = new SecurityManager() {
                private void check(Permission perm) {
                    // 禁止exec
                    if (perm instanceof java.io.FilePermission) {
                        String actions = perm.getActions();
                        if (actions != null && actions.contains("execute")) {
                            throw new SecurityException("execute denied!");
                        }
                    }
                    // 禁止设置新的SecurityManager，保护自己
                    if (perm instanceof java.lang.RuntimePermission) {
                        String name = perm.getName();
                        if (name != null && name.contains("setSecurityManager")) {
                            throw new SecurityException("System.setSecurityManager denied!");
                        }
                    }
                }

                @Override
                public void checkPermission(Permission perm) {
                    check(perm);
                }

                @Override
                public void checkPermission(Permission perm, Object context) {
                    check(perm);
                }
            };

            System.setSecurityManager(sm);
        }
```

只要在Java代码里简单加上面那一段，就可以禁止执行外部程序了。

## Java序列化的本身的蛋疼之处
其实Java自身的序列化机制就比较蛋疼，可以参考Effective Java里的。
http://allenlsy.com/NOTES-of-Effective-Java-10/


## 并非禁止外部程序执行，Java程序就安全了
要注意的是，如果可以任意执行Java代码，还可以做很多事情，比如写入ssh密钥，从而可以远程登陆，参考最近的Redis未授权访问漏洞：https://www.sebug.net/vuldb/ssvid-89715

## 总结

禁止JVM执行外部命令，是一个简单有效的提高JVM安全性的办法。但是以前没有见到有相关的内容，有点奇怪。
可以考虑在代码安全扫描时，加强对Runtime.exec相关代码的检测。
有些开源库喜欢用Runtime.exec来执行命令获取网卡mac等操作，个人表示相当的蛋疼，不会使用这样子的代码。