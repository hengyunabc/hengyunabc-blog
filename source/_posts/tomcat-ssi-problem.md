title: tomcat ssi配置及升级导致ssi include错误问题解决
date: 2015-04-12 20:26:01
tags:
 - tomcat
 - ssi


categories:
 - 技术

---

最近tomcat升级版本时，遇到了ssi解析的问题，记录下解决的过程，还有tomcat ssi配置的要点。

##tomcat 配置SSI的两种方式

Tomcat有两种方式支持SSI：Servlet和Filter。

###SSIServlet
通过Servlet，org.apache.catalina.ssi.SSIServlet，默认处理"*.shtml"的URL。

配置方式：

修改tomcat的 conf/web.xml文件，去掉下面配置的注释：
```xml
<servlet>
    <servlet-name>ssi</servlet-name>
    <servlet-class>
      org.apache.catalina.ssi.SSIServlet
    </servlet-class>
    <init-param>
      <param-name>buffered</param-name>
      <param-value>1</param-value>
    </init-param>
    <init-param>
      <param-name>debug</param-name>
      <param-value>0</param-value>
    </init-param>
    <init-param>
      <param-name>expires</param-name>
      <param-value>666</param-value>
    </init-param>
    <init-param>
      <param-name>isVirtualWebappRelative</param-name>
      <param-value>false</param-value>
    </init-param>
    <load-on-startup>4</load-on-startup>
</servlet>
 
<servlet-mapping>
    <servlet-name>ssi</servlet-name>
    <url-pattern>*.shtml</url-pattern>
</servlet-mapping>
```
###SSIFilter
通过Filter，org.apache.catalina.ssi.SSIFilter，默认处理"*.shtml"的URL。
 
配置方式：
 
修改tomcat的 conf/web.xml文件，打开去掉下面配置的注释：
```xml
<filter>
    <filter-name>ssi</filter-name>
    <filter-class>
      org.apache.catalina.ssi.SSIFilter
    </filter-class>
    <init-param>
      <param-name>contentType</param-name>
      <param-value>text/x-server-parsed-html(;.*)?</param-value>
    </init-param>
    <init-param>
      <param-name>debug</param-name>
      <param-value>0</param-value>
    </init-param>
    <init-param>
      <param-name>expires</param-name>
      <param-value>666</param-value>
    </init-param>
    <init-param>
      <param-name>isVirtualWebappRelative</param-name>
      <param-value>false</param-value>
    </init-param>
</filter>
 
<filter-mapping>
    <filter-name>ssi</filter-name>
    <url-pattern>*.shtml</url-pattern>
</filter-mapping>
```
###注意事项
**注意：两种配置方式最好不要同时打开，除非很清楚是怎样配置的。**

另外，在Tomcat的conf/context.xml里要配置privileged="true"，否则有些SSI特性不能生效。
```xml
<Context privileged="true">
```

##历史代码里处理SSI的办法
在公司的历史代码里，在一个公共的jar包里通过自定义一个EnhancedSSIServlet，继承了Tomcat的org.apache.catalina.ssi.SSIServlet来实现SSI功能的。

```java
@WebServlet(name="ssi",
            initParams={@WebInitParam(name="buffered", value="1"), @WebInitParam(name="debug", value="0"),
                        @WebInitParam(name="expires", value="666"), @WebInitParam(name="isVirtualWebappRelative", value="0"),
                        @WebInitParam(name="inputEncoding", value="UTF-8"), @WebInitParam(name="outputEncoding", value="UTF-8") },
            loadOnStartup=1, urlPatterns={"*.shtml"}, asyncSupported=true)
public class EnhancedSSIServlet extends SSIServlet {
```
其中**@WebServlet**是Servlet3.0规范里的，所以使用到web-common的web项目的web.xml文件都要配置为3.0版本以上，例如：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="3.0" xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
    http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">
 
</web-app>
```
**Tomcat是启动Web应用时，会扫描所有@WebServlet的类，并初始化。**

所以在使用到历史代码的项目都只能使用Tomcat服务器，并且不能在tomcat的conf/web.xml里打开SSI相关的配置。

##Tomcat版本升级的问题
Tomcat版本从7.0.57升级到7.0.59过程中，出现了无法解析SSI include指令的错误：
```java
SEVERE: #include--Couldn't include file: /pages/test/intelFilter.shtml
java.io.IOException: Couldn't get context for path: /pages/test/intelFilter.shtml
    at org.apache.catalina.ssi.SSIServletExternalResolver.getServletContextAndPathFromVirtualPath(SSIServletExternalResolver.java:422)
    at org.apache.catalina.ssi.SSIServletExternalResolver.getServletContextAndPath(SSIServletExternalResolver.java:465)
    at org.apache.catalina.ssi.SSIServletExternalResolver.getFileText(SSIServletExternalResolver.java:522)
    at org.apache.catalina.ssi.SSIMediator.getFileText(SSIMediator.java:161)
    at org.apache.catalina.ssi.SSIInclude.process(SSIInclude.java:50)
    at org.apache.catalina.ssi.SSIProcessor.process(SSIProcessor.java:159)
    at com.test.webcommon.servlet.EnhancedSSIServlet.processSSI(EnhancedSSIServlet.java:72)
    at org.apache.catalina.ssi.SSIServlet.requestHandler(SSIServlet.java:181)
    at org.apache.catalina.ssi.SSIServlet.doPost(SSIServlet.java:137)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:646)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:727)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:303)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:208)
    at org.apache.catalina.core.ApplicationDispatcher.invoke(ApplicationDispatcher.java:748)
    at org.apache.catalina.core.ApplicationDispatcher.doInclude(ApplicationDispatcher.java:604)
    at org.apache.catalina.core.ApplicationDispatcher.include(ApplicationDispatcher.java:543)
    at org.apache.jasper.runtime.JspRuntimeLibrary.include(JspRuntimeLibrary.java:954)
    at org.apache.jsp.pages.lottery.jczq.index_jsp._jspService(index_jsp.java:107)
    at org.apache.jasper.runtime.HttpJspBase.service(HttpJspBase.java:70)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:727)
    at org.apache.jasper.servlet.JspServletWrapper.service(JspServletWrapper.java:432)
    at org.apache.jasper.servlet.JspServlet.serviceJspFile(JspServlet.java:395)
    at org.apache.jasper.servlet.JspServlet.service(JspServlet.java:339)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:727)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:303)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:208)
```
仔细查看源代码后，发现不能处理的include指令代码如下：
```xml
<!--#include virtual="/pages/test/intelFilter.shtml"-->
```

经过对比调试Tomcat的代码，发现是在7.0.58版本时，改变了处理URL的方法，关键的处理函数是
```java
org.apache.catalina.core.ApplicationContext.getContext( String uri)
```
在7.0.57版本前，Tomcat在处理处理像/pages/test/intelFilter.shtml这样的路径时，恰好循环处理了"/"字符，使得childContext等于StandardContext，最终由StandardContext处理了/pages/test/intelFilter.shtml的请求。

**这个代码实际上是错误的，不过恰好处理了include virtual的情况。**

**在7.0.58版本修改了处理uri的代码，所以在升级Tomcat到7.0.59时出错了。**

7.0.57版的代码：
https://svn.apache.org/repos/asf/tomcat/tc7.0.x/tags/TOMCAT_7_0_57/java/org/apache/catalina/core/ApplicationContext.java
```java
/**
 * Return a <code>ServletContext</code> object that corresponds to a
 * specified URI on the server.  This method allows servlets to gain
 * access to the context for various parts of the server, and as needed
 * obtain <code>RequestDispatcher</code> objects or resources from the
 * context.  The given path must be absolute (beginning with a "/"),
 * and is interpreted based on our virtual host's document root.
 *
 * @param uri Absolute URI of a resource on the server
 */
@Override
public ServletContext getContext(String uri) {
    // Validate the format of the specified argument
    if ((uri == null) || (!uri.startsWith("/")))
        return (null);
    Context child = null;
    try {
        Host host = (Host) context.getParent();
        String mapuri = uri;
        while (true) {
            child = (Context) host.findChild(mapuri);
            if (child != null)
                break;
            int slash = mapuri.lastIndexOf('/');
            if (slash < 0)
                break;
            mapuri = mapuri.substring(0, slash);
        }
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        return (null);
    }
    if (child == null)
        return (null);
    if (context.getCrossContext()) {
        // If crossContext is enabled, can always return the context
        return child.getServletContext();
    } else if (child == context) {
        // Can still return the current context
        return context.getServletContext();
    } else {
        // Nothing to return
        return (null);
    }
}
```
7.0.58的代码：
https://svn.apache.org/repos/asf/tomcat/tc7.0.x/tags/TOMCAT_7_0_58/java/org/apache/catalina/core/ApplicationContext.java

那么正确的处理办法是怎样的？

仔细查看Tomcat的SSI配置的说明文档，发现有一个isVirtualWebappRelative的配置，而这个配置默认是false的。
```
isVirtualWebappRelative - Should "virtual" SSI directive paths be interpreted as relative to the context root, instead of the server root? Default false.
```
**也就是说，如果要支持“#include virtual="/b.shtml”绝对路径这种指令，就要配置isVirtualWebappRelative为true。
但是tomcat默认的SSI配置，以及上面的EnhancedSSIServlet类默认都配置isVirtualWebappRelative为false。**

因此，把EnhancedSSIServlet类里的isVirtualWebappRelative配置为true，重新测试，发现已经可以正常处理"#include virtual="/b.shtml"指令了。

相关的逻辑处理的代码在org.apache.catalina.ssi.SSIServletExternalResolver.getServletContextAndPathFromVirtualPath( String virtualPath)：
```java
protected ServletContextAndPath getServletContextAndPathFromVirtualPath(
         String virtualPath) throws IOException {
     if (!virtualPath.startsWith("/") && !virtualPath.startsWith("\\")) {
         return new ServletContextAndPath(context,
                 getAbsolutePath(virtualPath));
     }
     String normalized = RequestUtil.normalize(virtualPath);
     if (isVirtualWebappRelative) {
         return new ServletContextAndPath(context, normalized);
     }
     ServletContext normContext = context.getContext(normalized);
     if (normContext == null) {
         throw new IOException("Couldn't get context for path: "
                 + normalized);
     }
```

##总结
之前的EnhancedSSIServlet类的配置就不支持"#include virtual="/b.shtml"，这种绝对路径的SSI指令，而以前版本的Tomcat因为恰好处理了"/test.shtml"这种以"/"开头的url，因此以前版本的Tomcat没有报错。而升级后的Tomcat修正了代码，不再处理这种不合理的绝对路径请求了，所以报“ Couldn't get context for path”的异常。

把tomcat的ssi配置里的isVirtualWebappRelative设置为true就可以了。
 
最后，留一个小问题：

**tomcat是如何知道处理*.jsp请求的？是哪个servlet在起作用？**

##参考
https://tomcat.apache.org/tomcat-7.0-doc/ssi-howto.html