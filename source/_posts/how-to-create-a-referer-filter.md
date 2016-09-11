title: 怎样写一个RefererFilter
date: 2015-04-16 11:48:59
tags:
 - csrf
 - filter
 - referer

categories:
 - 技术

---
## 缘起
首先，用检查Referer的方式来防披御CSRF并不是很好的方法。因项目临时有需要，所以做为过渡方案。

为什么判断referer不是很好的办法？
- referer 可能为空
  - https跳转http没有referer
  - https跳转不同的域名的https没有referer
  - 通过特殊构造的POST请求没有referer
  - 一些的proxy会把referer去掉
  - 用户直接在浏览器里访问（GET请求）


* 判断的逻辑复杂（用正则匹配？）
* 友站中招，殃及池鱼
* 可以作为过渡方案，非长久之计

构造空referer请求的一些参考资料

- [Stripping Referrer for fun and profit](http://blog.kotowicz.net/2011/10/stripping-referrer-for-fun-and-profit.html)
- [Stripping the Referer in a Cross Domain POST request](http://webstersprodigy.net/2013/02/01/stripping-the-referer-in-a-cross-domain-post-request/)

防御CSRF目前比较好的办法是CSRF Token，参考另一篇blog：[Cookie & Session & CSRF](/cookie-and-session-and-csrf)。
##收集资料

先搜索下前人有没有这类相关的工作。
搜索到的关于RefererFilter的信息并不多。

不过这里学到了一些东东：
https://svn.apache.org/repos/asf/sling/tags/org.apache.sling.security-1.0.0/src/main/java/org/apache/sling/security/impl/ReferrerFilter.java
- 是否允许localhost, 127.0.0.1这样referer的请求？
- 是否允许本地的IP/host的请求？

再搜索下java里提取request的referer的方法，还有filter里重定向请求的方法。

再仔细看了下OWASP的文档：

https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)_Prevention_Cheat_Sheet

##确定方案
- 默认拦截“POST|PUT|DELETE|CONNECT|PATCH”的请求
- HttpServletRequest里提取到referer
- 用java.net.URL来提取referer里的host
- 判断host是否符合要求，支持完全匹配的域名和子域名
- 不符合要求的请求回应403或者重定向到指定的页面

为什么不用正则的方式来处理referer？
- 正则表达式通常比较慢
- 很难判断一个复杂的正则表达式是否真的正确
- URL是很复杂的，不要手动处理URL，参考[URL的语法](http://en.wikipedia.org/wiki/URI_scheme#Generic_syntax)

##思考需要提供的配置项
实际最终提供了这些配置项，考虑到像host这样的配置不是经常变动的，所以没有提供从外部配置文件加载配置的功能。
```plain
matchMethods   即拦截的方法，默认值"POST|PUT|DELETE|CONNECT|PATCH"，通常不用配置
allowSubDomainHosts 匹配子域名，以"|"分隔，如"test.com|abc.com"，
                     则http://test.com, http://xxx.test.com这样的请求都会匹配到，推荐优先使用这个配置
completeMatchHosts 完全匹配的域名，以"|"分隔，如"test.com|abc.com"，则只有http://test.com 这样的请求会匹配
                    像http://www.test.com 这样的请求不会被匹配
     
responseError  被拦截的请求的response的返回值，默认是403
redirectPath   被拦截的请求重定向到的url，如果配置了这个值，则会忽略responseError的配置。
                    比如可以配置重定向到自己定义的错误页： /referer_error.html
bAllowEmptyReferer  是否允许空referer，默认是false，除非很清楚，否则不要改动这个
bAllowLocalhost   是否允许localhost, 127.0.0.1 这样的referer的请求，默认是true，便于调试
bAllowAllIPAndHost  是否允许本机的所有IP和host的referer请求，默认是false
 ```

##编码的细节
* 重定向时，注意加上contextPath
  ```
  response.sendRedirect(request.getContextPath() + redirectPath);
  ```
 * 构造URL时，非法的URL会抛出RuntimeException，需要处理

##正确地处理URL
感觉这个有必要再次说明下：

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

##用curl来测试
最后用curl来做了一些测试：
```
curl  --header "Referer:http://test.com" http://localhost:8080/filter-test/referer
curl -X POST --header "Referer:http://test.com" http://localhost:8080/filter-test/referer
curl -X POST --header "Referer:xxxxx" http://localhost:8080/filter-test/referer
curl -X POST http://localhost:8080/filter-test/referer
curl -X POST --header "Referer:http://abc.test.com" http://localhost:8080/filter-test/referer
curl -X POST --header "Referer:http://abc.hello.com.test.com" http://localhost:8080/filter-test/referer
```

##实现的代码

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
 
/**
 * <pre>
 * 支持的配置项：
 * matchMethods   即拦截的方法，默认值"POST|PUT|DELETE|CONNECT|PATCH"，通常不用配置
 * allowSubDomainHosts 匹配子域名，以"|"分隔，如"test.com|abc.com"，
 *                     则http://test.com, http://xxx.test.com这样的请求都会匹配到，推荐优先使用这个配置
 * completeMatchHosts 完全匹配的域名，以"|"分隔，如"test.com|abc.com"，则只有http://test.com 这样的请求会匹配
 *                    像http://www.test.com 这样的请求不会被匹配
 *     
 * responseError  被拦截的请求的response的返回值，默认是403
 * redirectPath   被拦截的请求重定向到的url，如果配置了这个值，则会忽略responseError的配置。
 *                    比如可以配置重定向到自己定义的错误页： /referer_error.html
 * bAllowEmptyReferer  是否允许空referer，默认是false，除非很清楚，否则不要改动这个
 * bAllowLocalhost   是否允许localhost, 127.0.0.1 这样的referer的请求，默认是true，便于调试
 * bAllowAllIPAndHost  是否允许本机的所有IP和host的referer请求，默认是false
 *    
 * {@code
 * 	<filter>
 * 		<filter-name>refererFilter</filter-name>
 * 		<filter-class>com.test.RefererFilter</filter-class>
 * 		<init-param>
 * 			<param-name>completeMatchHosts</param-name>
 * 			<param-value>test.com|abc.com</param-value>
 * 		</init-param>
 * 		<init-param>
 * 			<param-name>allowSubDomainHosts</param-name>
 * 			<param-value>hello.com|xxx.yyy.com</param-value>
 * 		</init-param>
 * 	</filter>
 * 
 * 	<filter-mapping>
 * 		<filter-name>refererFilter</filter-name>
 * 		<url-pattern>/*</url-pattern>
 * 	</filter-mapping>
 * 	}
 * </pre>
 * 
 * @author hengyunabc
 *
 */
public class RefererFilter implements Filter {
	static final Logger logger = LoggerFactory.getLogger(RefererFilter.class);
	public static final String DEFAULT_MATHMETHODS = "POST|PUT|DELETE|CONNECT|PATCH";
 
	List<String> mathMethods = new ArrayList<>();
 
	boolean bAllowEmptyReferer = false;
 
	boolean bAllowLocalhost = true;
	boolean bAllowAllIPAndHost = false;
 
	/**
	 * when bAllowSubDomain is true, allowHosts is "test.com", then
	 * "www.test.com", "xxx.test.com" will be allow.
	 */
	boolean bAllowSubDomain = false;
 
	String redirectPath = null;
	int responseError = HttpServletResponse.SC_FORBIDDEN;
 
	HashSet<String> completeMatchHosts = new HashSet<String>();
 
	List<String> allowSubDomainHostList = new ArrayList<String>();
 
	@Override
	public void init(FilterConfig filterConfig) throws ServletException {
		mathMethods.addAll(getSplitStringList(filterConfig, "matchMethods", "\\|", DEFAULT_MATHMETHODS));
 
		completeMatchHosts.addAll(getSplitStringList(filterConfig, "completeMatchHosts", "\\|", ""));
 
		List<String> allowSubDomainHosts = getSplitStringList(filterConfig, "allowSubDomainHosts", "\\|", "");
		completeMatchHosts.addAll(allowSubDomainHosts);
		for (String host : allowSubDomainHosts) {
			// check the first char if is '.'
			if (!host.isEmpty() && host.charAt(0) != '.') {
				allowSubDomainHostList.add("." + host);
			} else {
				allowSubDomainHostList.add(host);
			}
		}
 
		responseError = getInt(filterConfig, "responseError", responseError);
		redirectPath = filterConfig.getInitParameter("redirectPath");
 
		bAllowEmptyReferer = getBoolean(filterConfig, "bAllowEmptyReferer", bAllowEmptyReferer);
 
		bAllowLocalhost = getBoolean(filterConfig, "bAllowLocalhost", bAllowLocalhost);
		if (bAllowLocalhost) {
			completeMatchHosts.add("localhost");
			completeMatchHosts.add("127.0.0.1");
			completeMatchHosts.add("[::1]");
		}
 
		bAllowAllIPAndHost = getBoolean(filterConfig, "bAllowAllIPAndHost", bAllowAllIPAndHost);
		if (bAllowAllIPAndHost) {
			completeMatchHosts.addAll(getAllIPAndHost());
		}
	}
 
	@Override
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException,
			ServletException {
		if (servletRequest instanceof HttpServletRequest && servletResponse instanceof HttpServletResponse) {
			HttpServletRequest request = (HttpServletRequest) servletRequest;
			HttpServletResponse response = (HttpServletResponse) servletResponse;
 
			String method = request.getMethod();
			/**
			 * if method not in POST|PUT|DELETE|CONNECT|PATCH, don't check
			 * referrer.
			 */
			if (!mathMethods.contains(method.trim().toUpperCase())) {
				filterChain.doFilter(request, response);
				return;
			}
 
			String referrer = request.getHeader("referer");
 
			boolean bAllow = false;
			if (isBlank(referrer)) {
				bAllow = bAllowEmptyReferer;
			} else {
				URL url = null;
				try {
					url = new URL(referrer);
					String host = url.getHost();
					if (completeMatchHosts.contains(host)) {
						bAllow = true;
					} else {
						for (String domain : allowSubDomainHostList) {
							if (host.endsWith(domain)) {
								bAllow = true;
								break;
							}
						}
					}
				} catch (RuntimeException e) {
					logger.error("illegal referrer! referrer: " + referrer, e);
					bAllow = false;
				}
			}
 
			if (bAllow) {
				filterChain.doFilter(request, response);
				return;
			} else {
				if (isBlank(redirectPath)) {
					response.sendError(HttpServletResponse.SC_FORBIDDEN);
				} else {
					response.sendRedirect(request.getContextPath() + redirectPath);
				}
			}
		} else {
			filterChain.doFilter(servletRequest, servletResponse);
		}
	}
 
	@Override
	public void destroy() {
 
	}
 
	private static boolean isBlank(CharSequence cs) {
		int strLen;
		if (cs == null || (strLen = cs.length()) == 0) {
			return true;
		}
		for (int i = 0; i < strLen; i++) {
			if (Character.isWhitespace(cs.charAt(i)) == false) {
				return false;
			}
		}
		return true;
	}
 
	private static boolean getBoolean(FilterConfig filterConfig, String parameter, boolean defaultParameterValue) {
		String parameterString = filterConfig.getInitParameter(parameter);
		if (parameterString == null) {
			return defaultParameterValue;
		}
		return Boolean.parseBoolean(parameterString.trim());
	}
 
	private static int getInt(FilterConfig filterConfig, String parameter, int defaultParameterValue) {
		String parameterString = filterConfig.getInitParameter(parameter);
		if (parameterString == null) {
			return defaultParameterValue;
		}
		return Integer.parseInt(parameterString.trim());
	}
 
	/**
	 * <pre>
	 * getSplitStringList(filterConfig, "hosts", "\\|", "test.com|abc.com");
	 * 
	 * if hosts is "hello.com|google.com", will return {"hello.com", google.com"}.
	 * if hosts is null, will return {"test.com", "abc.com"}
	 * </pre>
	 * 
	 * @param filterConfig
	 * @param parameter
	 * @param regex
	 * @param defaultParameterValue
	 * @return
	 */
	private static List<String> getSplitStringList(FilterConfig filterConfig, String parameter, String regex, String defaultParameterValue) {
		String parameterString = filterConfig.getInitParameter(parameter);
		if (parameterString == null) {
			parameterString = defaultParameterValue;
		}
 
		String[] split = parameterString.split("\\|");
		if (split != null) {
			List<String> resultList = new LinkedList<String>();
			for (String method : split) {
				resultList.add(method.trim());
			}
			return resultList;
		}
		return Collections.emptyList();
	}
 
	public static Set<String> getAllIPAndHost() {
		HashSet<String> resultSet = new HashSet<String>();
 
		Enumeration<NetworkInterface> interfaces;
		try {
			interfaces = NetworkInterface.getNetworkInterfaces();
			while (interfaces.hasMoreElements()) {
				NetworkInterface nic = interfaces.nextElement();
				Enumeration<InetAddress> addresses = nic.getInetAddresses();
				while (addresses.hasMoreElements()) {
					InetAddress address = addresses.nextElement();
					if (address instanceof Inet4Address) {
						resultSet.add(address.getHostAddress());
						resultSet.add(address.getHostName());
					} else if (address instanceof Inet6Address) {
						// TODO how to process Inet6Address?
						// resultSet.add("[" + address.getHostAddress() + "]");
						// resultSet.add(address.getHostName());
					}
				}
			}
		} catch (SocketException e) {
			logger.error("getAllIPAndHost error!", e);
		}
		return resultSet;
	}
}
```

##其它的一些东东
在浏览器里如何访问IPV6的地址？

用"[]"把IPV6地址包围起来，比如localhost的：
```
http://[::1]
```

参考：

http://superuser.com/questions/367780/how-to-connect-a-website-has-only-ipv6-address-without-domain-name

https://msdn.microsoft.com/en-us/library/windows/desktop/ms740593(v=vs.85).aspx