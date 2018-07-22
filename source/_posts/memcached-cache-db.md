---
title: 应对Memcached缓存失效，导致高并发查询DB的几种思路
date: 2014-05-03 05:58:28
tags:
 - memcached
 - cache


categories:
  - 技术

---


## update 

最近看到nginx的合并回源，这个和下面的思路有点像。不过nginx的思路还是在控制缓存失效时的并发请求，而不是当缓存快要失效时，及时地更新缓存。

nginx合并回源，参考：http://blog.csdn.net/brainkick/article/details/8570698

update: 2015-04-23

## 前言

当Memcached缓存失效时，容易出现高并发的查询DB，导致DB压力骤然上升。

这篇blog主要是探讨如何在缓存将要失效时，及时地更新缓存，而不是如何在缓存失效之后，如何防止高并发的DB查询。

**个人认为，当缓存将要失效时，及时地把新的数据刷到memcached里，这个是解决缓存失效瞬间高并发查DB的最好方法。那么如何及时地知道缓存将要失效？**

解决这个问题有几种思路：

比如一个key是aaa，失效时间是30s。

### 定期从DB里查询数据，再刷到memcached里

这种方法有个缺点是，有些业务的key可能是变化的，不确定的。

而且不好界定哪些数据是应该查询出来放到缓存中的，难以区分冷热数据。

### 当缓存取到为null时，加锁去查询DB，只允许一个线程去查询DB

这种方式不太靠谱，不多讨论。而且如果是多个web服务器的话，还是有可能有并发的操作。

### 在向memcached写入value时，同时写入当前机器在时间作为过期时间

当get得到数据时，如果当前时间 - 过期时间 > 5s，则后台启动一个任务去查询DB，更新缓存。

当然，这里的后台任务必须保证同一个key，只有一个线程在执行查询DB的任务，不然这个还是高并发查询DB。

缺点是要把过期时间和value合在一起序列化，取出数据后，还要反序列化。很不方便。



网上大部分文章提到的都是前面两种方式，有少数文章提到第3种方式。下面提出一种基于两个key的方法：

### 两个key，一个key用来存放数据，另一个用来标记失效时间

比如key是aaa，设置失效时间为30s，则另一个key为expire_aaa，失效时间为25s。

在取数据时，用multiget，同时取出aaa和expire_aaa，如果expire_aaa的value == null，则后台启动一个任务去查询DB，更新缓存。和上面类似。



对于后台启动一个任务去查询DB，更新缓存，要保证一个key只有一个线程在执行，这个如何实现？

对于同一个进程，简单加锁即可。拿到锁的就去更新DB，没拿到锁的直接返回。



对于集群式的部署的，如何实现只允许一个任务执行？

这里就要用到memcached的add命令了。

**add命令是如果不存在key，则设置成功，返回true，如果已存在key，则不存储，返回false。**

当get expired_aaa是null时，则add expired_aaa 过期时间由自己灵活处理。比如设置为3秒。

如果成功了，再去查询DB，查到数据后，再set expired_aaa为25秒。set aaa 为30秒。

综上所述，来梳理下流程：

比如一个key是aaa，失效时间是30s。查询DB在1s内。

* put数据时，设置aaa过期时间30s，设置expire_aaa过期时间25s；
* get数据时，multiget  aaa 和 expire_aaa，如果expired_aaa对应的value != null，则直接返回aaa对应的数据给用户。如果expire_aaa返回value == null，则后台启动一个任务，尝试add expire_aaa，并设置超时过间为3s。这里设置为3s是为了防止后台任务失败或者阻塞，如果这个任务执行失败，那么3秒后，如果有另外的用户访问，那么可以再次尝试查询DB。如果add执行成功，则查询DB，再更新aaa的缓存，并设置expire_aaa的超时时间为25s。 

###  时间存到Value里，再结合add命令来保证只有一个线程去刷新数据

update:2014-06-29

最近重新思考了下这个问题。发现第4种两个key的办法比较耗memcached的内存，因为key数翻倍了。结合第3种方式，重新设计了下，思路如下：

* 仍然使用两个key的方案：key 和 __load_{key}

    其中，__load_{key} 这个key相当于一个锁，只允许add成功的线程去更新数据，而这个key的超时时间是比较短的，不会一直占用memcached的内存。

* 在set 到Memcached的value中，加上一个时间，(time, value)，time是memcached上的key未来会过期的时间，并不是当前系统时间。
* 当get到数据时，检查时间是否快要超时： time - now < 5 * 1000，假定设置了快要超时的时间是5秒。
* 如果是，则后台启动一个新的线程：

    * 尝试 add __load_{key}，
	* 如果成功，则去加载新的数据，并set到memcached中。
	* 原来的线程直接返回value给调用者。


按上面的思路，用xmemcached封装了下：

DataLoader，用户要实现的加载数据的回调接口：

```java
public interface DataLoader {
	public <T> T load();
}
```

RefreshCacheManager，用户只需要关心这这两个接口函数：

```java
public class RefreshCacheManager {
	static public <T> T tryGet(MemcachedClient memcachedClient, final String key, final int expire, final DataLoader dataLoader);
	static public <T> T autoRetryGet(MemcachedClient memcachedClient, final String key, final int expire, final DataLoader dataLoader);
}
```

其中autoRetryGet函数如果get到是null，内部会自动重试4次，每次间隔500ms。

RefreshCacheManager内部自动处理数据快过期，重新刷新到memcached的逻辑。

详细的封装代码在这里：https://gist.github.com/hengyunabc/cc57478bfcb4cd0553c2

## 总结

我个人是倾向于第5种方式的，因为很简单，直观。**比第4种方式要节省内存，而且不用mget，在使用memcached集群时不用担心出麻烦事。**

这种两个key的方式，还有一个好处，就是数据是自然冷热适应的。如果是冷数据，30秒都没有人访问，那么数据会过期。

如果是热门数据，一直有大流量访问，那么数据就是一直热的，而且数据一直不会过期。