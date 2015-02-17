layout: post
title: netstat统计的tcp连接数与⁄proc⁄pid⁄fd下socket类型fd数量不一致的分析
date: 2015-01-14 10:26:37
tags:
- netstat
- jvm
- gc
- socket

categories:
 - java
 - 网络

---
最近，线上一个应用，发现socket数缓慢增长，并且不回收，超过警告线之后，被运维监控自动重启了。

首先到zabbix上观察JVM历史记录，发现JVM-Perm space最近两周没有数据，猜测是程序从JDK7切换到JDK8了。问过开发人员之后，程序已经很久没有重启了，最近才重新发布的。而在这期间，线上的Java运行环境已经从JDK7升级到JDK8了。

因为jdk8里没有Perm space了，换成了Metaspace。

###netstat
到线上服务器上，用netstat来统计进程的connection数量。
```bash
netstat -antp | grep pid | wc -l
```
发现比zabbix上的统计socket数量要少100多，netstat统计只有100多，而zabbix上监控数据有300多。

于是到/proc/$pid/fd下统计socket类型的fd数量：
```bash
cd /proc/$pid/fd
ls -al | grep socket | wc -l
```
发现数据和zabbix上的数据一致。

###netstat是怎么统计的
####下载netstat的源代码
http://unix.stackexchange.com/questions/21503/source-code-of-netstat 
```bash
apt-get source net-tools
```
从netstat的代码里，大概可以看到是读取/proc/net/tcp里面的数据来获取统计信息的。
####java和c版的简单netstat的实现
java版的

http://www.cs.earlham.edu/~jeremiah/LinuxSocket.java

C版的：

http://www.netmite.com/android/mydroid/system/core/toolbox/netstat.c

####用starce跟踪netstat
```
strace netstat -antp 
```
可以发现netstat把/proc 下的很多数据都读取出来了。于是大致可以知道netstat是把/proc/pid/fd 下面的数据和/proc/net/下面的数据汇总，对照得到统计结果的。

####哪些socket会没有被netstat统计到？
又在网上找了下，发现这里有说到socket如果创建了，没有bind或者connect，就不会被netstat统计到。

http://serverfault.com/questions/153983/sockets-found-by-lsof-but-not-by-netstat

实际上，也就是如果socket创建了，没有被使用，那么就只会在/proc/pid/fd下面有，而不会在/proc/net/下面有相关数据。

简单测试了下，的确是这样：
```c
int socket = socket(PF_INET,SOCK_STREAM,0); //不使用
```

另外，即使socket是使用过的，如果执行shutdown后，刚开始里，用netstat可以统计到socket的状态是FIN_WAIT1。过一段时间，netstat统计不到socket的信息的，但是在/proc/pid/fd下，还是可以找到。

中间的时候，自己写了个程序，把/proc/pid/fd 下的inode和/proc/net/下面的数据比较，发现的确有些socket的inode不会出现在/proc/net/下。
####用lsof查看
用lsof查看socket inode：

###触发GC，回收socket
于是尝试触发GC，看下socket会不会被回收：
```bash
jmap -histo:live <pid>
```
结果，发现socket都被回收了。

再看下AbstractPlainSocketImpl的finalize方法：
```java
    /**
     * Cleans up if the user forgets to close it.
     */
    protected void finalize() throws IOException {
        close();
    }
```
可以看到socket是会在GC时，被close掉的。
写个程序来测试下：
```java
public class TestServer {
	public static void main(String[] args) throws IOException, InterruptedException {
		for(int i = 0; i < 10; ++i){
			ServerSocket socket = new ServerSocket(i + 10000);
			System.err.println(socket);
		}
		System.in.read();
	}
}
```
先执行，查看/proc/pid/fd，可以发现有相关的socket fd，再触发GC，可以发现socket被回收掉了。
##其它的东东
####anon_inode:[eventpoll]
```
ls -al /proc/pid/fd
```
可以看到有像这样的输出：
```
661 -> anon_inode:[eventpoll]
```
这种类型的inode，是epoll创建的。

再扯远一点，linux下java里的selector实现是epoll结合一个pipe来实现事件通知功能的。所以在NIO程序里，会有anon_inode:[eventpoll]和pipe类型的fd。
####为什么tail -f /proc/$pid/fd/1 不能读取到stdout的数据
http://unix.stackexchange.com/questions/152773/why-cant-i-tail-f-proc-pid-fd-1
##总结
原因是jdk升级之后，GC的工作方式有变化，FullGC执行的时间变长了，导致有些空闲的socket没有被回收。

本文比较乱，记录下一些工具和技巧。