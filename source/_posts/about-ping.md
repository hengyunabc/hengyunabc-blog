---
title: 扯谈网络编程之自己实现ping
date: 2014-05-03 05:58:28
tags:
 - ping
 - network
 - udp
 - ICMP


categories:
  - 技术

---


## 前言

ping是基于ICMP（Internet Control Message Protocol）协议实现的，而ICMP协议是在IP层实现的。

ping实际上是发起者发送一个Echo Request(type = 8)的，远程主机回应一个Echo Reply(type = 0)的过程。

## 为什么用ping不能测试某一个端口

刚开始接触网络的时候，可能很多人都有疑问，怎么用ping来测试远程主机的某个特定端口？

其实如果看下ICMP协议，就可以发现ICMP里根本没有端口这个概念，也就根本无法实现测试某一个端口了。

ICMP协议的包格式可以直接参考wiki： https://en.wikipedia.org/wiki/Ping_(networking_utility)#ICMP_packet

## Ping如何计算请问耗时

在ping命令的输出上，可以看到有显示请求的耗时，那么这个耗时是怎么得到的呢？

```
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=6.28 ms
```

从Echo Request的格式里，看到不时间相关的东东，但是因为是Echo，即远程主机会原样返回Data数据，所以Ping的发起方把时间放到了Data数据里，当得到Echo Reply里，取到发送时间，再和当前时间比较，就可以得到耗时了。当然，还有其它的思路，比如记录每一个包的发送时间，当得到返回时，再计算得到时间差，但显然这样的实现太复杂了。

## Ping如何区分不同的进程？

我们都知道本机IP，远程IP，本机端口，远程端口，四个元素才可以确定唯的一个信道。而ICMP里没有端口，那么一个ping程序如何知道哪些包才是发给自己的？或者说操作系统如何区别哪个Echo Reply是要发给哪个进程的？

**实际上操作系统不能区别，所有的本机IP，远程IP相同的ICMP程序都可以接收到同一份数据。**

程序自己要根据Identifier来区分到底一个ICMP包是不是发给自己的。在Linux下，Ping发出去的Echo Request包里Identifier就是进程pid，远程主机会返回一个Identifier相同的Echo Reply包。

可以接下面的方法简单验证：

启动系统自带的ping程序，查看其pid。

设定自己实现的ping程序的identifier为上面得到的pid，然后发Echo Request包。

可以发现系统ping程序会接收到远程主机的回应。

## 自己实现ping

自己实现ping要用到rawsocket，在linux下需要root权限。网上有很多实现的程序，但是有很多地方不太对的。自己总结实现了一个（最好用g++编绎）：

```cpp
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <errno.h>
#include <netinet/ip.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/time.h>
 
unsigned short csum(unsigned short *ptr, int nbytes) {
	register long sum;
	unsigned short oddbyte;
	register short answer;
 
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
 
inline double countMs(timeval before, timeval after){
	return (after.tv_sec - before.tv_sec)*1000 + (after.tv_usec - before.tv_usec)/1000.0;
}
 
#pragma pack(1)
struct EchoPacket {
	u_int8_t type;
	u_int8_t code;
	u_int16_t checksum;
	u_int16_t identifier;
	u_int16_t sequence;
	timeval timestamp;
	char data[40];   //sizeof(EchoPacket) == 64
};
#pragma pack()
 
void ping(in_addr_t source, in_addr_t destination) {
	static int sequence = 1;
	static int pid = getpid();
	static int ipId = 0;
 
	char sendBuf[sizeof(iphdr) + sizeof(EchoPacket)] = { 0 };
 
	struct iphdr* ipHeader = (iphdr*)sendBuf;
	ipHeader->version = 4;
	ipHeader->ihl = 5;
 
	ipHeader->tos = 0;
	ipHeader->tot_len = htons(sizeof(sendBuf));
 
	ipHeader->id = htons(ipId++);
	ipHeader->frag_off = htons(0x4000);  //set Flags: don't fragment
 
	ipHeader->ttl = 64;
	ipHeader->protocol = IPPROTO_ICMP;
	ipHeader->check = 0;
	ipHeader->saddr = source;
	ipHeader->daddr = destination;
 
	ipHeader->check = csum((unsigned short*)ipHeader, ipHeader->ihl * 2);
 
	EchoPacket* echoRequest = (EchoPacket*)(sendBuf + sizeof(iphdr));
	echoRequest->type = 8;
	echoRequest->code = 0;
	echoRequest->checksum = 0;
	echoRequest->identifier = htons(pid);
	echoRequest->sequence = htons(sequence++);
	gettimeofday(&(echoRequest->timestamp), NULL);
	u_int16_t ccsum = csum((unsigned short*)echoRequest, sizeof(sendBuf) - sizeof(iphdr));
 
	echoRequest->checksum = ccsum;
 
	struct sockaddr_in sin;
	sin.sin_family = AF_INET;
	sin.sin_port = htons(0);
	sin.sin_addr.s_addr = destination;
 
	int s = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);
	if (s == -1) {
		perror("socket");
		return;
	}
 
	//IP_HDRINCL to tell the kernel that headers are included in the packet
	if (setsockopt(s, IPPROTO_IP, IP_HDRINCL, "1",sizeof("1")) < 0) {
		perror("Error setting IP_HDRINCL");
		exit(0);
	}
 
	sendto(s, sendBuf, sizeof(sendBuf), 0, (struct sockaddr *) &sin, sizeof(sin));
 
	char responseBuf[sizeof(iphdr) + sizeof(EchoPacket)] = {0};
 
	struct sockaddr_in receiveAddress;
	socklen_t len = sizeof(receiveAddress);
	int reveiveSize = recvfrom(s, (void*)responseBuf, sizeof(responseBuf), 0, (struct sockaddr *) &receiveAddress, &len);
 
	if(reveiveSize == sizeof(responseBuf)){
		EchoPacket* echoResponse = (EchoPacket*) (responseBuf + sizeof(iphdr));
		//TODO check identifier == pid ?
		if(echoResponse->type == 0){
			struct timeval tv;
			gettimeofday(&tv, NULL);
 
			in_addr tempAddr;
			tempAddr.s_addr = destination;
			printf("%d bytes from %s : icmp_seq=%d ttl=%d time=%.2f ms\n",
					sizeof(EchoPacket),
					inet_ntoa(tempAddr),
					ntohs(echoResponse->sequence),
					((iphdr*)responseBuf)->ttl,
					countMs(echoResponse->timestamp, tv));
		}else{
			printf("response error, type:%d\n", echoResponse->type);
		}
	}else{
		printf("error, response size != request size.\n");
	}
 
	close(s);
}
 
int main(void) {
	in_addr_t source = inet_addr("192.168.1.100");
	in_addr_t destination = inet_addr("192.168.1.1");
	for(;;){
		ping(source, destination);
		sleep(1);
	}
 
	return 0;
}
```

## 安全相关的一些东东

* 死亡之Ping  http://zh.wikipedia.org/wiki/%E6%AD%BB%E4%BA%A1%E4%B9%8BPing

尽管是很老的漏洞，但是也可以看出协议栈的实现也不是那么的靠谱。

* Ping flood   http://en.wikipedia.org/wiki/Ping_flood

服务器关闭ping服务，默认是0,是开启：

```
echo 1 > /proc/sys/net/ipv4/icmp_echo_ignore_all
```

## 总结

在自己实现的过程中，发现有一些蛋疼的地方，如

协议文档不够清晰，得反复对照；

有时候一个小地方处理不对，很难查bug，即使程序能正常工作，但也并不代表它是正确的；

用wireshark可以很方便验证自己写的程序有没有问题。

## 参考

* http://en.wikipedia.org/wiki/Ping_(networking_utility)

* http://en.wikipedia.org/wiki/ICMP_Destination_Unreachable

* http://tools.ietf.org/pdf/rfc792.pdf