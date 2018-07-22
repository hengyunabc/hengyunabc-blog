---
title: spring boot应用启动原理分析
date: 2015-11-30 16:19:38
tags:
 - spring
 - spring-boot
 - ClassLoader

categories:
 - 技术
---



## 前言

本文分析的是spring boot 1.3.* 的工作原理。spring boot 1.4.* 之后打包结构发现了变化，增加了`BOOT-INF`目录，但是基本原理还是不变的。

关于spring boot 1.4.* 里ClassLoader的变化，可以参考：http://hengyunabc.github.io/spring-boot-classloader/



## spring boot quick start
在spring boot里，很吸引人的一个特性是可以直接把应用打包成为一个jar/war，然后这个jar/war是可以直接启动的，不需要另外配置一个Web Server。

如果之前没有使用过spring boot可以通过下面的demo来感受下。
下面以这个工程为例，演示如何启动Spring boot项目：

```bash
git clone git@github.com:hengyunabc/spring-boot-demo.git
mvn spring-boot-demo
java -jar target/demo-0.0.1-SNAPSHOT.jar
```
如果使用的IDE是spring sts或者idea，可以通过向导来创建spring boot项目。

也可以参考官方教程：
http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#getting-started-first-application

## 对spring boot的两个疑问

刚开始接触spring boot时，通常会有这些疑问

* spring boot如何启动的？
* spring boot embed tomcat是如何工作的？ 静态文件，jsp，网页模板这些是如何加载到的？


下面来分析spring boot是如何做到的。

## 打包为单个jar时，spring boot的启动方式
maven打包之后，会生成两个jar文件：

```
demo-0.0.1-SNAPSHOT.jar
demo-0.0.1-SNAPSHOT.jar.original
```
其中demo-0.0.1-SNAPSHOT.jar.original是默认的maven-jar-plugin生成的包。

demo-0.0.1-SNAPSHOT.jar是spring boot maven插件生成的jar包，里面包含了应用的依赖，以及spring boot相关的类。下面称之为fat jar。

先来查看spring boot打好的包的目录结构（不重要的省略掉）：

```
├── META-INF
│   ├── MANIFEST.MF
├── application.properties
├── com
│   └── example
│       └── SpringBootDemoApplication.class
├── lib
│   ├── aopalliance-1.0.jar
│   ├── spring-beans-4.2.3.RELEASE.jar
│   ├── ...
└── org
    └── springframework
        └── boot
            └── loader
                ├── ExecutableArchiveLauncher.class
                ├── JarLauncher.class
                ├── JavaAgentDetector.class
                ├── LaunchedURLClassLoader.class
                ├── Launcher.class
                ├── MainMethodRunner.class
                ├── ...                
```

依次来看下这些内容。

### MANIFEST.MF

```
Manifest-Version: 1.0
Start-Class: com.example.SpringBootDemoApplication
Implementation-Vendor-Id: com.example
Spring-Boot-Version: 1.3.0.RELEASE
Created-By: Apache Maven 3.3.3
Build-Jdk: 1.8.0_60
Implementation-Vendor: Pivotal Software, Inc.
Main-Class: org.springframework.boot.loader.JarLauncher
```

可以看到有Main-Class是org.springframework.boot.loader.JarLauncher ，这个是jar启动的Main函数。

还有一个Start-Class是com.example.SpringBootDemoApplication，这个是我们应用自己的Main函数。

```java
@SpringBootApplication
public class SpringBootDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootDemoApplication.class, args);
    }
}
```

### com/example 目录
这下面放的是应用的.class文件。

### lib目录
这里存放的是应用的Maven依赖的jar包文件。
比如spring-beans，spring-mvc等jar。

### org/springframework/boot/loader 目录
这下面存放的是Spring boot loader的.class文件。


## Archive的概念

* archive即归档文件，这个概念在linux下比较常见
* 通常就是一个tar/zip格式的压缩包
* jar是zip格式

在spring boot里，抽象出了Archive的概念。

一个archive可以是一个jar（JarFileArchive），也可以是一个文件目录（ExplodedArchive）。可以理解为Spring boot抽象出来的统一访问资源的层。

上面的demo-0.0.1-SNAPSHOT.jar 是一个Archive，然后demo-0.0.1-SNAPSHOT.jar里的/lib目录下面的每一个Jar包，也是一个Archive。

```java
public abstract class Archive {
	public abstract URL getUrl();
	public String getMainClass();
	public abstract Collection<Entry> getEntries();
	public abstract List<Archive> getNestedArchives(EntryFilter filter);
```
可以看到Archive有一个自己的URL，比如：

```
jar:file:/tmp/target/demo-0.0.1-SNAPSHOT.jar!/
```
还有一个getNestedArchives函数，这个实际返回的是demo-0.0.1-SNAPSHOT.jar/lib下面的jar的Archive列表。它们的URL是：

```
jar:file:/tmp/target/demo-0.0.1-SNAPSHOT.jar!/lib/aopalliance-1.0.jar
jar:file:/tmp/target/demo-0.0.1-SNAPSHOT.jar!/lib/spring-beans-4.2.3.RELEASE.jar
```

## JarLauncher

从MANIFEST.MF可以看到Main函数是JarLauncher，下面来分析它的工作流程。

JarLauncher类的继承结构是：

```java
class JarLauncher extends ExecutableArchiveLauncher
class ExecutableArchiveLauncher extends Launcher
```

### 以demo-0.0.1-SNAPSHOT.jar创建一个Archive：

JarLauncher先找到自己所在的jar，即demo-0.0.1-SNAPSHOT.jar的路径，然后创建了一个Archive。

下面的代码展示了如何从一个类找到它的加载的位置的技巧：

```java
	protected final Archive createArchive() throws Exception {
		ProtectionDomain protectionDomain = getClass().getProtectionDomain();
		CodeSource codeSource = protectionDomain.getCodeSource();
		URI location = (codeSource == null ? null : codeSource.getLocation().toURI());
		String path = (location == null ? null : location.getSchemeSpecificPart());
		if (path == null) {
			throw new IllegalStateException("Unable to determine code source archive");
		}
		File root = new File(path);
		if (!root.exists()) {
			throw new IllegalStateException(
					"Unable to determine code source archive from " + root);
		}
		return (root.isDirectory() ? new ExplodedArchive(root)
				: new JarFileArchive(root));
	}
```

### 获取lib/下面的jar，并创建一个LaunchedURLClassLoader

JarLauncher创建好Archive之后，通过getNestedArchives函数来获取到demo-0.0.1-SNAPSHOT.jar/lib下面的所有jar文件，并创建为List<Archive>。

注意上面提到，Archive都是有自己的URL的。

获取到这些Archive的URL之后，也就获得了一个URL[]数组，用这个来构造一个自定义的ClassLoader：LaunchedURLClassLoader。

创建好ClassLoader之后，再从MANIFEST.MF里读取到Start-Class，即com.example.SpringBootDemoApplication，然后创建一个新的线程来启动应用的Main函数。

```java
	/**
	 * Launch the application given the archive file and a fully configured classloader.
	 */
	protected void launch(String[] args, String mainClass, ClassLoader classLoader)
			throws Exception {
		Runnable runner = createMainMethodRunner(mainClass, args, classLoader);
		Thread runnerThread = new Thread(runner);
		runnerThread.setContextClassLoader(classLoader);
		runnerThread.setName(Thread.currentThread().getName());
		runnerThread.start();
	}

	/**
	 * Create the {@code MainMethodRunner} used to launch the application.
	 */
	protected Runnable createMainMethodRunner(String mainClass, String[] args,
			ClassLoader classLoader) throws Exception {
		Class<?> runnerClass = classLoader.loadClass(RUNNER_CLASS);
		Constructor<?> constructor = runnerClass.getConstructor(String.class,
				String[].class);
		return (Runnable) constructor.newInstance(mainClass, args);
	}
```


### LaunchedURLClassLoader

LaunchedURLClassLoader和普通的URLClassLoader的不同之处是，它提供了从Archive里加载.class的能力。

结合Archive提供的getEntries函数，就可以获取到Archive里的Resource。当然里面的细节还是很多的，下面再描述。


## spring boot应用启动流程总结

看到这里，可以总结下Spring Boot应用的启动流程：


1. spring boot应用打包之后，生成一个fat jar，里面包含了应用依赖的jar包，还有Spring  boot loader相关的类
2. Fat jar的启动Main函数是JarLauncher，它负责创建一个LaunchedURLClassLoader来加载/lib下面的jar，并以一个新线程启动应用的Main函数。


## spring boot loader里的细节

代码地址：https://github.com/spring-projects/spring-boot/tree/master/spring-boot-tools/spring-boot-loader

### JarFile URL的扩展
Spring boot能做到以一个fat jar来启动，最重要的一点是它实现了jar in jar的加载方式。

JDK原始的JarFile URL的定义可以参考这里：

http://docs.oracle.com/javase/7/docs/api/java/net/JarURLConnection.html

原始的JarFile URL是这样子的：

```
jar:file:/tmp/target/demo-0.0.1-SNAPSHOT.jar!/
```
jar包里的资源的URL：
```
jar:file:/tmp/target/demo-0.0.1-SNAPSHOT.jar!/com/example/SpringBootDemoApplication.class
```
可以看到对于Jar里的资源，定义以'!/'来分隔。原始的JarFile URL只支持一个'!/'。

Spring boot扩展了这个协议，让它支持多个'!/'，就可以表示jar in jar，jar in directory的资源了。

比如下面的URL表示demo-0.0.1-SNAPSHOT.jar这个jar里lib目录下面的spring-beans-4.2.3.RELEASE.jar里面的MANIFEST.MF：

```
jar:file:/tmp/target/demo-0.0.1-SNAPSHOT.jar!/lib/spring-beans-4.2.3.RELEASE.jar!/META-INF/MANIFEST.MF
```

#### 自定义URLStreamHandler，扩展JarFile和JarURLConnection

在构造一个URL时，可以传递一个Handler，而JDK自带有默认的Handler类，应用可以自己注册Handler来处理自定义的URL。

```java
public URL(String protocol,
           String host,
           int port,
           String file,
           URLStreamHandler handler)
    throws MalformedURLException
```
参考：
https://docs.oracle.com/javase/8/docs/api/java/net/URL.html#URL-java.lang.String-java.lang.String-int-java.lang.String-

Spring boot通过注册了一个自定义的Handler类来处理多重jar in jar的逻辑。

这个Handler内部会用SoftReference来缓存所有打开过的JarFile。

在处理像下面这样的URL时，会循环处理'!/'分隔符，从最上层出发，先构造出demo-0.0.1-SNAPSHOT.jar这个JarFile，再构造出spring-beans-4.2.3.RELEASE.jar这个JarFile，然后再构造出指向MANIFEST.MF的JarURLConnection。

```
jar:file:/tmp/target/demo-0.0.1-SNAPSHOT.jar!/lib/spring-beans-4.2.3.RELEASE.jar!/META-INF/MANIFEST.MF
```

```java
//org.springframework.boot.loader.jar.Handler
public class Handler extends URLStreamHandler {
	private static final String SEPARATOR = "!/";
	private static SoftReference<Map<File, JarFile>> rootFileCache;
	@Override
	protected URLConnection openConnection(URL url) throws IOException {
		if (this.jarFile != null) {
			return new JarURLConnection(url, this.jarFile);
		}
		try {
			return new JarURLConnection(url, getRootJarFileFromUrl(url));
		}
		catch (Exception ex) {
			return openFallbackConnection(url, ex);
		}
	}
	public JarFile getRootJarFileFromUrl(URL url) throws IOException {
		String spec = url.getFile();
		int separatorIndex = spec.indexOf(SEPARATOR);
		if (separatorIndex == -1) {
			throw new MalformedURLException("Jar URL does not contain !/ separator");
		}
		String name = spec.substring(0, separatorIndex);
		return getRootJarFile(name);
	}
```

#### ClassLoader如何读取到Resource

对于一个ClassLoader，它需要哪些能力？

* 查找资源
* 读取资源

对应的API是：

```java
public URL findResource(String name)
public InputStream getResourceAsStream(String name)
```


上面提到，Spring boot构造LaunchedURLClassLoader时，传递了一个URL[]数组。数组里是lib目录下面的jar的URL。

对于一个URL，JDK或者ClassLoader如何知道怎么读取到里面的内容的？

实际上流程是这样子的：

* LaunchedURLClassLoader.loadClass
* URL.getContent()
* URL.openConnection()
* Handler.openConnection(URL)

最终调用的是JarURLConnection的getInputStream()函数。

```java
//org.springframework.boot.loader.jar.JarURLConnection
	@Override
	public InputStream getInputStream() throws IOException {
		connect();
		if (this.jarEntryName.isEmpty()) {
			throw new IOException("no entry name specified");
		}
		return this.jarEntryData.getInputStream();
	}
```

从一个URL，到最终读取到URL里的内容，整个过程是比较复杂的，总结下：

* spring boot注册了一个Handler来处理"jar:"这种协议的URL
* spring boot扩展了JarFile和JarURLConnection，内部处理jar in jar的情况
* 在处理多重jar in jar的URL时，spring boot会循环处理，并缓存已经加载到的JarFile
* 对于多重jar in jar，实际上是解压到了临时目录来处理，可以参考JarFileArchive里的代码
* 在获取URL的InputStream时，最终获取到的是JarFile里的JarEntryData

这里面的细节很多，只列出比较重要的一些点。

然后，URLClassLoader是如何getResource的呢？

URLClassLoader在构造时，有URL[]数组参数，它内部会用这个数组来构造一个URLClassPath:

```java
URLClassPath ucp = new URLClassPath(urls);
```
在 URLClassPath 内部会为这些URLS 都构造一个Loader，然后在getResource时，会从这些Loader里一个个去尝试获取。
如果获取成功的话，就像下面那样包装为一个Resource。

```java
Resource getResource(final String name, boolean check) {
    final URL url;
    try {
        url = new URL(base, ParseUtil.encodePath(name, false));
    } catch (MalformedURLException e) {
        throw new IllegalArgumentException("name");
    }
    final URLConnection uc;
    try {
        if (check) {
            URLClassPath.check(url);
        }
        uc = url.openConnection();
        InputStream in = uc.getInputStream();
        if (uc instanceof JarURLConnection) {
            /* Need to remember the jar file so it can be closed
             * in a hurry.
             */
            JarURLConnection juc = (JarURLConnection)uc;
            jarfile = JarLoader.checkJar(juc.getJarFile());
        }
    } catch (Exception e) {
        return null;
    }
    return new Resource() {
        public String getName() { return name; }
        public URL getURL() { return url; }
        public URL getCodeSourceURL() { return base; }
        public InputStream getInputStream() throws IOException {
            return uc.getInputStream();
        }
        public int getContentLength() throws IOException {
            return uc.getContentLength();
        }
    };
}
```
从代码里可以看到，实际上是调用了url.openConnection()。这样完整的链条就可以连接起来了。

注意，URLClassPath这个类的代码在JDK里没有自带，在这里看到 http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7u40-b43/sun/misc/URLClassPath.java#506

### 在IDE/开放目录启动Spring boot应用
在上面只提到在一个fat jar里启动Spring boot应用的过程，下面分析IDE里Spring boot是如何启动的。

在IDE里，直接运行的Main函数是应用自己的Main函数：

```java
@SpringBootApplication
public class SpringBootDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootDemoApplication.class, args);
    }
}
```
其实在IDE里启动Spring boot应用是最简单的一种情况，因为依赖的Jar都让IDE放到classpath里了，所以Spring boot直接启动就完事了。

还有一种情况是在一个开放目录下启动Spring boot启动。所谓的开放目录就是把fat jar解压，然后直接启动应用。

```
java org.springframework.boot.loader.JarLauncher
```

这时，Spring boot会判断当前是否在一个目录里，如果是的，则构造一个ExplodedArchive（前面在jar里时是JarFileArchive），后面的启动流程类似fat jar的。


### Embead Tomcat的启动流程

#### 判断是否在web环境

spring boot在启动时，先通过一个简单的查找Servlet类的方式来判断是不是在web环境：

```java
private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
    "org.springframework.web.context.ConfigurableWebApplicationContext" };

private boolean deduceWebEnvironment() {
    for (String className : WEB_ENVIRONMENT_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return false;
        }
    }
    return true;
}
```
如果是的话，则会创建AnnotationConfigEmbeddedWebApplicationContext，否则Spring context就是AnnotationConfigApplicationContext：

```java
//org.springframework.boot.SpringApplication
	protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
		if (contextClass == null) {
			try {
				contextClass = Class.forName(this.webEnvironment
						? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Unable create a default ApplicationContext, "
								+ "please specify an ApplicationContextClass",
						ex);
			}
		}
		return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
	}
```

#### 获取EmbeddedServletContainerFactory的实现类
spring boot通过获取EmbeddedServletContainerFactory来启动对应的web服务器。

常用的两个实现类是TomcatEmbeddedServletContainerFactory和JettyEmbeddedServletContainerFactory。

启动Tomcat的代码：

```java
//TomcatEmbeddedServletContainerFactory
@Override
public EmbeddedServletContainer getEmbeddedServletContainer(
        ServletContextInitializer... initializers) {
    Tomcat tomcat = new Tomcat();
    File baseDir = (this.baseDirectory != null ? this.baseDirectory
            : createTempDir("tomcat"));
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    Connector connector = new Connector(this.protocol);
    tomcat.getService().addConnector(connector);
    customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    tomcat.getEngine().setBackgroundProcessorDelay(-1);
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
    }
    prepareContext(tomcat.getHost(), initializers);
    return getTomcatEmbeddedServletContainer(tomcat);
}
```

会为tomcat创建一个临时文件目录，如：
/tmp/tomcat.2233614112516545210.8080，做为tomcat的basedir。里面会放tomcat的临时文件，比如work目录。

还会初始化Tomcat的一些Servlet，比如比较重要的default/jsp servlet：

```java
private void addDefaultServlet(Context context) {
    Wrapper defaultServlet = context.createWrapper();
    defaultServlet.setName("default");
    defaultServlet.setServletClass("org.apache.catalina.servlets.DefaultServlet");
    defaultServlet.addInitParameter("debug", "0");
    defaultServlet.addInitParameter("listings", "false");
    defaultServlet.setLoadOnStartup(1);
    // Otherwise the default location of a Spring DispatcherServlet cannot be set
    defaultServlet.setOverridable(true);
    context.addChild(defaultServlet);
    context.addServletMapping("/", "default");
}

private void addJspServlet(Context context) {
    Wrapper jspServlet = context.createWrapper();
    jspServlet.setName("jsp");
    jspServlet.setServletClass(getJspServletClassName());
    jspServlet.addInitParameter("fork", "false");
    jspServlet.setLoadOnStartup(3);
    context.addChild(jspServlet);
    context.addServletMapping("*.jsp", "jsp");
    context.addServletMapping("*.jspx", "jsp");
}
```

### spring boot的web应用如何访问Resource
当spring boot应用被打包为一个fat jar时，是如何访问到web resource的？

实际上是通过Archive提供的URL，然后通过Classloader提供的访问classpath resource的能力来实现的。

#### index.html

比如需要配置一个index.html，这个可以直接放在代码里的src/main/resources/static目录下。

对于index.html欢迎页，spring boot在初始化时，就会创建一个ViewController来处理：

```java
//ResourceProperties
public class ResourceProperties implements ResourceLoaderAware {

	private static final String[] SERVLET_RESOURCE_LOCATIONS = { "/" };

	private static final String[] CLASSPATH_RESOURCE_LOCATIONS = {
			"classpath:/META-INF/resources/", "classpath:/resources/",
			"classpath:/static/", "classpath:/public/" };

```
```java
//WebMvcAutoConfigurationAdapter
		@Override
		public void addViewControllers(ViewControllerRegistry registry) {
			Resource page = this.resourceProperties.getWelcomePage();
			if (page != null) {
				logger.info("Adding welcome page: " + page);
				registry.addViewController("/").setViewName("forward:index.html");
			}
		}
```

#### template

像页面模板文件可以放在src/main/resources/template目录下。但这个实际上是模板的实现类自己处理的。比如ThymeleafProperties类里的：

```java
public static final String DEFAULT_PREFIX = "classpath:/templates/";
```

#### jsp
jsp页面和template类似。实际上是通过spring mvc内置的JstlView来处理的。

可以通过配置spring.view.prefix来设定jsp页面的目录：

```
spring.view.prefix: /WEB-INF/jsp/
```

### spring boot里统一的错误页面的处理

对于错误页面，Spring boot也是通过创建一个BasicErrorController来统一处理的。

```java
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController

```
对应的View是一个简单的HTML提醒：

```java
	@Configuration
	@ConditionalOnProperty(prefix = "server.error.whitelabel", name = "enabled", matchIfMissing = true)
	@Conditional(ErrorTemplateMissingCondition.class)
	protected static class WhitelabelErrorViewConfiguration {

		private final SpelView defaultErrorView = new SpelView(
				"<html><body><h1>Whitelabel Error Page</h1>"
						+ "<p>This application has no explicit mapping for /error, so you are seeing this as a fallback.</p>"
						+ "<div id='created'>${timestamp}</div>"
						+ "<div>There was an unexpected error (type=${error}, status=${status}).</div>"
						+ "<div>${message}</div></body></html>");

		@Bean(name = "error")
		@ConditionalOnMissingBean(name = "error")
		public View defaultErrorView() {
			return this.defaultErrorView;
		}
```
spring boot的这个做法很好，避免了传统的web应用来出错时，默认抛出异常，容易泄密。

### spring boot应用的maven打包过程

先通过maven-shade-plugin生成一个包含依赖的jar，再通过spring-boot-maven-plugin插件把spring boot loader相关的类，还有MANIFEST.MF打包到jar里。

### spring boot里有颜色日志的实现
当在shell里启动spring boot应用时，会发现它的logger输出是有颜色的，这个特性很有意思。

可以通过这个设置来关闭：

```
spring.output.ansi.enabled=false
```

原理是通过AnsiOutputApplicationListener ，这个来获取这个配置，然后设置logback在输出时，加了一个 ColorConverter，通过org.springframework.boot.ansi.AnsiOutput ，对一些字段进行了渲染。



## 一些代码小技巧

#### 实现ClassLoader时，支持JDK7并行加载
可以参考LaunchedURLClassLoader里的LockProvider

```java
public class LaunchedURLClassLoader extends URLClassLoader {

	private static LockProvider LOCK_PROVIDER = setupLockProvider();
	private static LockProvider setupLockProvider() {
		try {
			ClassLoader.registerAsParallelCapable();
			return new Java7LockProvider();
		}
		catch (NoSuchMethodError ex) {
			return new LockProvider();
		}
	}

	@Override
	protected Class<?> loadClass(String name, boolean resolve)
			throws ClassNotFoundException {
		synchronized (LaunchedURLClassLoader.LOCK_PROVIDER.getLock(this, name)) {
			Class<?> loadedClass = findLoadedClass(name);
			if (loadedClass == null) {
				Handler.setUseFastConnectionExceptions(true);
				try {
					loadedClass = doLoadClass(name);
				}
				finally {
					Handler.setUseFastConnectionExceptions(false);
				}
			}
			if (resolve) {
				resolveClass(loadedClass);
			}
			return loadedClass;
		}
	}
```

#### 检测jar包是否通过agent加载的

InputArgumentsJavaAgentDetector，原理是检测jar的URL是否有"-javaagent:"的前缀。

```java
private static final String JAVA_AGENT_PREFIX = "-javaagent:";
```

#### 获取进程的PID

ApplicationPid，可以获取PID。

```java
	private String getPid() {
		try {
			String jvmName = ManagementFactory.getRuntimeMXBean().getName();
			return jvmName.split("@")[0];
		}
		catch (Throwable ex) {
			return null;
		}
	}
```

#### 包装Logger类

spring boot里自己包装了一套logger，支持java, log4j, log4j2, logback，以后有需要自己包装logger时，可以参考这个。

在org.springframework.boot.logging包下面。

#### 获取原始启动的main函数

通过堆栈里获取的方式，判断main函数，找到原始启动的main函数。

```java
private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
```


## spirng boot的一些缺点：
当spring boot应用以一个fat jar方式运行时，会遇到一些问题。以下是个人看法：

* 日志不知道放哪，默认是输出到stdout的
* 数据目录不知道放哪, jenkinns的做法是放到 ${user.home}/.jenkins 下面
* 相对目录API不能使用，servletContext.getRealPath("/") 返回的是NULL
* spring boot应用喜欢把配置都写到代码里，有时会带来混乱。一些简单可以用xml来表达的配置可能会变得难读，而且凌乱。


## 总结

spring boot通过扩展了jar协议，抽象出Archive概念，和配套的JarFile，JarUrlConnection，LaunchedURLClassLoader，从而实现了上层应用无感知的all in one的开发体验。尽管Executable war并不是spring提出的概念，但spring boot让它发扬光大。

spring boot是一个惊人的项目，可以说是spring的第二春，spring-cloud-config, spring-session, metrics, remote shell等都是深爱开发者喜爱的项目、特性。几乎可以肯定设计者是有丰富的一线开发经验，深知开发人员的痛点。
