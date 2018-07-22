---
title: 深入Spring Boot：实现对Fat Jar jsp的支持
date: 2018-05-16 19:45:28
tags:
 - spring
 - spring-boot
 - java
 - tomcat

categories:
 - 技术
---


## spring boot 对于jsp支持的限制

对于jsp的支持，Spring Boot官方只支持了war的打包方式，不支持fat jar。参考官方文档： https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-web-applications.html#boot-features-jsp-limitations

这里spring boot官方说是tomcat的问题，实际上是spring boot自己改变了打包格式引起的。参考之前的文章：http://hengyunabc.github.io/spring-boot-classloader/#spring-boot-1-3-%E5%92%8C-1-4-%E7%89%88%E6%9C%AC%E7%9A%84%E5%8C%BA%E5%88%AB

原来的结构之下，tomcat是可以扫描到fat jar里的`META-INF/resources`目录下面的资源的。在增加了`BOOT-INF/classes`之后，则tomcat扫描不到了。

那么怎么解决这个问题呢？下面给出一种方案，来实现对spring boot fat jar/exploded directory的jsp的支持。

## 个性化配置tomcat，把BOOT-INF/classes 加入tomcat的ResourceSet

在tomcat里，所有扫描到的资源都会放到所谓的`ResourceSet`里。比如servlet 3规范里的应用jar包的`META-INF/resources`就是一个`ResourceSet`。

现在需要想办法把spring boot打出来的fat jar的`BOOT-INF/classes`目录加到`ResourceSet`里。

下面通过实现tomcat的 `LifecycleListener`接口，在`Lifecycle.CONFIGURE_START_EVENT`事件里，获取到`BOOT-INF/classes`的URL，再把这个URL加入到`WebResourceSet`里。

```java
/**
 * Add main class fat jar/exploded directory into tomcat ResourceSet.
 *
 * @author hengyunabc 2017-07-29
 *
 */
public class StaticResourceConfigurer implements LifecycleListener {

	private final Context context;

	StaticResourceConfigurer(Context context) {
		this.context = context;
	}

	@Override
	public void lifecycleEvent(LifecycleEvent event) {
		if (event.getType().equals(Lifecycle.CONFIGURE_START_EVENT)) {
			URL location = this.getClass().getProtectionDomain().getCodeSource().getLocation();

			if (ResourceUtils.isFileURL(location)) {
				// when run as exploded directory
				String rootFile = location.getFile();
				if (rootFile.endsWith("/BOOT-INF/classes/")) {
					rootFile = rootFile.substring(0, rootFile.length() - "/BOOT-INF/classes/".length() + 1);
				}
				if (!new File(rootFile, "META-INF" + File.separator + "resources").isDirectory()) {
					return;
				}

				try {
					location = new File(rootFile).toURI().toURL();
				} catch (MalformedURLException e) {
					throw new IllegalStateException("Can not add tomcat resources", e);
				}
			}

			String locationStr = location.toString();
			if (locationStr.endsWith("/BOOT-INF/classes!/")) {
				// when run as fat jar
				locationStr = locationStr.substring(0, locationStr.length() - "/BOOT-INF/classes!/".length() + 1);
				try {
					location = new URL(locationStr);
				} catch (MalformedURLException e) {
					throw new IllegalStateException("Can not add tomcat resources", e);
				}
			}
			this.context.getResources().createWebResourceSet(ResourceSetType.RESOURCE_JAR, "/", location,
					"/META-INF/resources");

		}
	}
}
```

为了让spring boot embedded tomcat加载这个 `StaticResourceConfigurer`，还需要一个`EmbeddedServletContainerCustomizer`的配置：

```java
@Configuration
@ConditionalOnProperty(name = "tomcat.staticResourceCustomizer.enabled", matchIfMissing = true)
public class TomcatConfiguration {
	@Bean
	public EmbeddedServletContainerCustomizer staticResourceCustomizer() {
		return new EmbeddedServletContainerCustomizer() {
			@Override
			public void customize(ConfigurableEmbeddedServletContainer container) {
				if (container instanceof TomcatEmbeddedServletContainerFactory) {
					((TomcatEmbeddedServletContainerFactory) container)
							.addContextCustomizers(new TomcatContextCustomizer() {
								@Override
								public void customize(Context context) {
									context.addLifecycleListener(new StaticResourceConfigurer(context));
								}
							});
				}
			}

		};
	}
}
```

这样子的话，spring boot就可以支持fat jar里的jsp资源了。

demo地址： https://github.com/hengyunabc/spring-boot-fat-jar-jsp-sample

## 总结
* spring boot改变了打包结构，导致tomcat没有办法扫描到fat jar里的`/BOOT-INF/classes`
* 通过一个`StaticResourceConfigurer`把fat jar里的`/BOOT-INF/classes`加到tomcat的`ResourceSet`来解决问题

