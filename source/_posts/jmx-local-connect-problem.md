---
title: 通过Heap dump排查Java JMX连接不上的问题
date: 2019-06-11 01:01:41
tags:
 - java
 - jvm
 - jmx
 - visualvm

categories:
 - 技术
---


### 背景

最近排查一个JMX本地连接问题，记录一下。

我们的启动脚本在应用启动后，会通过JMX来动态检查应用状态，那么这里就需要动态启动JMX功能了。

### 动态打开Java进程的JMX端口

1. 通过下面的代码，可以动态的让目标进程加载`management-agent`
2. 打开JMX功能后，通过获取`com.sun.management.jmxremote.localConnectorAddress`的Agent Property，可以获取到JMX URL

```java
    public MBeanServerConnection connect(String pid) throws IOException {
        String address = attachJmx(pid);

        JMXServiceURL serviceURL = new JMXServiceURL(address);
        connector = JMXConnectorFactory.connect(serviceURL);
        return connector.getMBeanServerConnection();
    }

    private String attachJmx(String pid) throws IOException {
        try {
            virtualmachine = VirtualMachine.attach(pid);
        } catch (AttachNotSupportedException e) {
            throw new IOException(e);
        }
        String javaHome = virtualmachine.getSystemProperties().getProperty("java.home");
        String agentPath = javaHome + File.separator + "jre" + File.separator + "lib" + File.separator
                           + "management-agent.jar";
        File file = new File(agentPath);
        if (!file.exists()) {
            agentPath = javaHome + File.separator + "lib" + File.separator + "management-agent.jar";
            file = new File(agentPath);
            if (!file.exists()) {
                throw new IOException("Management agent not found");
            }
        }

        agentPath = file.getCanonicalPath();
        try {
            virtualmachine.loadAgent(agentPath, "com.sun.management.jmxremote");
        } catch (AgentLoadException e) {
            throw new IOException(e);
        } catch (AgentInitializationException agentinitializationexception) {
            throw new IOException(agentinitializationexception);
        }

        Properties properties = virtualmachine.getAgentProperties();
        String address = (String) properties.get("com.sun.management.jmxremote.localConnectorAddress");
        virtualmachine.detach();
        return address;
    }
```

### 为什么JMX连接会失败？

在用上面的代码动态去连接目标进程时，抛出了下面的异常：

```
java.rmi.ConnectException: Connection refused to host: 11.164.235.11; nested exception is:
  java.net.ConnectException: Connection refused
  at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:619)
  at sun.rmi.transport.tcp.TCPChannel.createConnection(TCPChannel.java:216)
  at sun.rmi.transport.tcp.TCPChannel.newConnection(TCPChannel.java:202)
  at sun.rmi.server.UnicastRef.invoke(UnicastRef.java:130)
  at java.rmi.server.RemoteObjectInvocationHandler.invokeRemoteMethod(RemoteObjectInvocationHandler.java:227)
  at java.rmi.server.RemoteObjectInvocationHandler.invoke(RemoteObjectInvocationHandler.java:179)
  at com.sun.proxy.$Proxy0.newClient(Unknown Source)
  at javax.management.remote.rmi.RMIConnector.getConnection(RMIConnector.java:2430)
  at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:308)
  at javax.management.remote.JMXConnectorFactory.connect(JMXConnectorFactory.java:270)
  at javax.management.remote.JMXConnectorFactory.connect(JMXConnectorFactory.java:229)
  at com.test.jmx.JmxLocalConnector.connect(JmxLocalConnector.java:28)
Caused by: java.net.ConnectException: Connection refused
  at java.net.PlainSocketImpl.socketConnect(Native Method)
  at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
  at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
  at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
  at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
  at java.net.Socket.connect(Socket.java:589)
  at java.net.Socket.connect(Socket.java:538)
  at java.net.Socket.<init>(Socket.java:434)
  at java.net.Socket.<init>(Socket.java:211)
  at sun.rmi.transport.proxy.RMIDirectSocketFactory.createSocket(RMIDirectSocketFactory.java:40)
  at sun.rmi.transport.proxy.RMIMasterSocketFactory.createSocket(RMIMasterSocketFactory.java:148)
  at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:613)
  ... 13 more
```


* 检查本机IP是 `11.164.234.171`
* 为什么rmi连接的是一个外部的IP `11.164.235.11`？

通过调试，发现`management-agent`加载成功了，`localConnectorAddress`的值是：

```
jmx:rmi://127.0.0.1/stub/rO0ABXN9AAAAAQAlamF2YXgubWFuYWdlbWVudC5yZW1vdGUucm1pLlJNSVNlcnZlcnhyABdqYXZhLmxhbmcucmVmbGVjdC5Qcm94eeEn2iDMEEPLAgABTAABaHQAJUxqYXZhL2xhbmcvcmVmbGVjdC9JbnZvY2F0aW9uSGFuZGxlcjt4cHNyAC1qYXZhLnJtaS5zZXJ2ZXIuUmVtb3RlT2JqZWN0SW52b2NhdGlvbkhhbmRsZXIAAAAAAAAAAgIAAHhyABxqYXZhLnJtaS5zZXJ2ZXIuUmVtb3RlT2JqZWN002G0kQxhMx4DAAB4cHc4AAtVbmljYXN0UmVmMgAADTExLjE2NC4yMzUuMTEAAIfoCEScYyGQodFlwEdFAAABawK/zE6AAQB4
```

**为什么显示的是`127.0.0.1`，但实际连接的是`11.164.235.11`？是不是在连接时出的问题？**

再仔细调试，发现

1. jmx是获取到stub后的字符串
2. 做base64解密，再通过`ObjectInputStream`解析
3. `readObject`得到`RMIServer`对象来连接的。

```java
//javax.management.remote.rmi.RMIConnector.findRMIServer(JMXServiceURL, Map<String, Object>)

    //--------------------------------------------------------------------
    // Private stuff - RMIServer creation
    //--------------------------------------------------------------------

    private RMIServer findRMIServer(JMXServiceURL directoryURL,
            Map<String, Object> environment)
            throws NamingException, IOException {
        final boolean isIiop = RMIConnectorServer.isIiopURL(directoryURL,true);
        if (isIiop) {
            // Make sure java.naming.corba.orb is in the Map.
            environment.put(EnvHelp.DEFAULT_ORB,resolveOrb(environment));
        }

        String path = directoryURL.getURLPath();
        int end = path.indexOf(';');
        if (end < 0) end = path.length();
        if (path.startsWith("/jndi/"))
            return findRMIServerJNDI(path.substring(6,end), environment, isIiop);
        else if (path.startsWith("/stub/"))
            return findRMIServerJRMP(path.substring(6,end), environment, isIiop);
        else if (path.startsWith("/ior/")) {
            if (!IIOPHelper.isAvailable())
                throw new IOException("iiop protocol not available");
            return findRMIServerIIOP(path.substring(5,end), environment, isIiop);
        } else {
            final String msg = "URL path must begin with /jndi/ or /stub/ " +
                    "or /ior/: " + path;
            throw new MalformedURLException(msg);
        }
    }

    private RMIServer findRMIServerJRMP(String base64, Map<String, ?> env, boolean isIiop)
        throws IOException {
        // could forbid "iiop:" URL here -- but do we need to?
        final byte[] serialized;
        try {
            serialized = base64ToByteArray(base64);
        } catch (IllegalArgumentException e) {
            throw new MalformedURLException("Bad BASE64 encoding: " +
                    e.getMessage());
        }
        final ByteArrayInputStream bin = new ByteArrayInputStream(serialized);

        final ClassLoader loader = EnvHelp.resolveClientClassLoader(env);
        final ObjectInputStream oin =
                (loader == null) ?
                    new ObjectInputStream(bin) :
                    new ObjectInputStreamWithLoader(bin, loader);
        final Object stub;
        try {
            stub = oin.readObject();
        } catch (ClassNotFoundException e) {
            throw new MalformedURLException("Class not found: " + e);
        }
        return (RMIServer)stub;
    }
```

通过代码处理，发现

```
rO0ABXN9AAAAAQAlamF2YXgubWFuYWdlbWVudC5yZW1vdGUucm1pLlJNSVNlcnZlcnhyABdqYXZhLmxhbmcucmVmbGVjdC5Qcm94eeEn2iDMEEPLAgABTAABaHQAJUxqYXZhL2xhbmcvcmVmbGVjdC9JbnZvY2F0aW9uSGFuZGxlcjt4cHNyAC1qYXZhLnJtaS5zZXJ2ZXIuUmVtb3RlT2JqZWN0SW52b2NhdGlvbkhhbmRsZXIAAAAAAAAAAgIAAHhyABxqYXZhLnJtaS5zZXJ2ZXIuUmVtb3RlT2JqZWN002G0kQxhMx4DAAB4cHc4AAtVbmljYXN0UmVmMgAADTExLjE2NC4yMzUuMTEAAIfoCEScYyGQodFlwEdFAAABawK/zE6AAQB4
```

转换为了：

```
RMIServerImpl_Stub[UnicastRef2 [liveRef: [endpoint:[11.164.235.11:26449](remote),objID:[-5ddae53d:16b0887d710:-7fff, 7209064096623493021]]]]
```

**可见RMI Server的IP的确是`11.164.235.11`。**

那么现在问题变成了：

* 为什么JVM动态加载了`management-agent`，得到的JMX URL是指向外部IP的？

### 通过heap dump定位IP字符串

但是调试`management-agent`的加载过程可能会比较痛苦，于是考虑从别的地方入手。

从上面的调查里，发现`management-agent`启动之后，`11.164.235.11`这个外部IP就会出现在JVM内存里，那么考虑用heap dump的方式来定位。

通过执行heap dump，再用`jvisualvm`来分析。

用OQL来搜索所有包含`11.164.235.11`的String：

```sql
select s from java.lang.String s where s.toString().equals("11.164.235.11")
```

可以发现有好几个结果：

![](https://user-images.githubusercontent.com/1683936/58687979-bde91880-83b5-11e9-88f8-ee498e22337f.png)

再依次点开，查看引用，**发现其中一个引用的字段名是`localHost`**：

![](https://user-images.githubusercontent.com/1683936/58688109-1c15fb80-83b6-11e9-8afb-cfbedfd0f036.jpg)

**因此可以猜测：是不是localHost域名解析有问题？**

### 定位localHost域名解析问题

执行hostname命令，得到机器名，再ping一下：

```
$hostname
web-app201641.we42

$ping web-app201641.we42
PING web-app201641.we42 (11.164.235.11) 56(84) bytes of data.
```

发现本机被解析到`11.164.235.11`了，但是本机的IP是`11.164.234.171`：

```
$ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 11.164.234.171  netmask 255.255.255.0  broadcast 11.164.234.255
```

到这里，大概猜到原因了，检查下 `/etc/hosts`文件，果然发现有配置：

```
11.164.235.11  web-app201641.we42
```

把这个错误的host配置去掉之后，再执行jmx连接终于成功了。

为什么会有错误的hosts配置呢？据说是机器迁移时遗留的。

### 总结

动态JMX连接的工作原理：

1. 让目标`VirtualMachine`动态加载`management-agent`
2. 从Agent Properties里获取到JMX连接地址：`com.sun.management.jmxremote.localConnectorAddress`
3. JMX URL里带`stub`的字符串，实际上是base64转换为byte[]，再用`ObjectInputStream`转换为`RMIServer`
4. JMX实际上是通过RMI来连接的


排查问题的关键：

1. 定位错误连接的IP
2. heap dump
3. 用OQL从heap dump里查找IP字符串，再查看相关的引用来获取信息

### 链接

* [ViauslVM](https://visualvm.github.io/)
* [Object Query Language (OQL)](http://cr.openjdk.java.net/~sundar/8022483/webrev.01/raw_files/new/src/share/classes/com/sun/tools/hat/resources/oqlhelp.html)