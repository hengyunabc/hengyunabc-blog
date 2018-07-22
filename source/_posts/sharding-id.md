---
title: 分片(Sharding)的全局ID生成
date: 2014-02-16 16:58:28
tags:
 - redis
 - sharding


categories:
  - 技术

---

这里最后redis生成ID的文章已经过时，新的请参考： http://blog.csdn.net/hengyunabc/article/details/44244951


## 前言

数据在分片时，典型的是分库分表，就有一个全局ID生成的问题。单纯的生成全局ID并不是什么难题，但是生成的ID通常要满足分片的一些要求：

* 不能有单点故障。
* 以时间为序，或者ID里包含时间。这样一是可以少一个索引，二是冷热数据容易分离。
* 可以控制ShardingId。比如某一个用户的文章要放在同一个分片内，这样查询效率高，修改也容易。
* 不要太长，最好64bit。使用long比较好操作，如果是96bit，那就要各种移位相当的不方便，还有可能有些组件不能支持这么大的ID。

先来看看老外的做法，以时间顺序：

## flickr

flickr巧妙地使用了mysql的自增ID，及replace into语法，十分简洁地实现了分片ID生成功能。

首先，创建一个表：

```sql
CREATE TABLE `Tickets64` (
  `id` bigint(20) unsigned NOT NULL auto_increment,
  `stub` char(1) NOT NULL default '',
  PRIMARY KEY  (`id`),
  UNIQUE KEY `stub` (`stub`)
) ENGINE=MyISAM

```

使用上面的sql可以得到一个ID：

```sql
REPLACE INTO Tickets64 (stub) VALUES ('a');
SELECT LAST_INSERT_ID();
```

因为使用了replace into的语法，实际上，Tickets64这个表里的数据永远都是这样的：

```
+-------------------+------+
| id                | stub |
+-------------------+------+
| 72157623227190423 |    a |
+-------------------+------+
```

那么如何解决单点故障呢？
很简单，利用mysql的自增ID即可。比如有两台ID生成服务器，设置成下面即可：

```
TicketServer1:
auto-increment-increment = 2
auto-increment-offset = 1
 
TicketServer2:
auto-increment-increment = 2
auto-increment-offset = 2
```

优点：

* 简单可靠。

缺点：

* ID只是一个ID，没有带入时间，shardingId等信息。

## twitter

twitter利用zookeeper实现了一个全局ID生成的服务snowflake，https://github.com/twitter/snowflake，可以生成全局唯一的64bit ID。

生成的ID的构成：

```
时间--用前面41 bit来表示时间，精确到毫秒，可以表示69年的数据
机器ID--用10 bit来表示，也就是说可以部署1024台机器
序列数--用12 bit来表示，意味着每台机器，每毫秒最多可以生成4096个ID
```

优点：

* 充分把信息保存到ID里。

缺点：

* 结构略复杂，要依赖zookeeper。
* 分片ID不能灵活生成。

## instagram

instagram参考了flickr的方案，再结合twitter的经验，利用Postgres数据库的特性，实现了一个更简单可靠的ID生成服务。

instagram是这样设计它们的ID的：

* 使用41 bit来存放时间，精确到毫秒，可以使用41年。
* 使用13 bit来存放逻辑分片ID。
* 使用10 bit来存放自增长ID，意味着每台机器，每毫秒最多可以生成1024个ID

以instagram举的例子为说明：

假定时间是September 9th, 2011, at 5:00pm，则毫秒数是1387263000（直接使用系统得到的从1970年开始的毫秒数）。那么先把时间数据放到ID里：

`id = 1387263000 << (64-41)`

再把分片ID放到时间里，假定用户ID是31341，有2000个逻辑分片，则分片ID是31341 % 2000 -> 1341：

`id |= 1341 << (64-41-13)`

最后，把自增序列放ID里，假定前一个序列是5000,则新的序列是5001：

`id |= (5001 % 1024)`

这样就得到了一个全局的分片ID。

下面列出instagram使用的Postgres schema的sql：

```sql
REATE OR REPLACE FUNCTION insta5.next_id(OUT result bigint) AS $$
DECLARE
    our_epoch bigint := 1314220021721;
    seq_id bigint;
    now_millis bigint;
    shard_id int := 5;
BEGIN
    SELECT nextval('insta5.table_id_seq') %% 1024 INTO seq_id;
 
    SELECT FLOOR(EXTRACT(EPOCH FROM clock_timestamp()) * 1000) INTO now_millis;
    result := (now_millis - our_epoch) << 23;
    result := result | (shard_id << 10);
    result := result | (seq_id);
END;
$$ LANGUAGE PLPGSQL;
```

则在插入新数据时，直接用类似下面的SQL即可（连请求生成ID的步骤都省略了！）：

```sql
CREATE TABLE insta5.our_table (
    "id" bigint NOT NULL DEFAULT insta5.next_id(),
    ...rest of table schema...
)
```

即使是不懂Postgres数据库，也能从上面的SQL看出个大概。把这个移植到mysql上应该也不是什么难事。

缺点：

* 貌似真的没啥缺点。

优点：

* 充分把信息保存到ID里。

* 充分利用数据库自身的机制，程序完全不用额外处理，直接插入到对应的分片的表即可。

## 使用redis的方案

站在前人的肩膀上，我想到了一个利用redis + lua的方案。

首先，lua内置的时间函数不能精确到毫秒，因此先要修改下redis的代码，增加currentMiliseconds函数，我偷懒，直接加到math模块里了。

修改redis代码下的scripting.c文件，加入下面的内容：

```cpp
#include <sys/time.h>
 
int redis_math_currentMiliseconds (lua_State *L);
 
void scriptingInit(void) {
    ...
    lua_pushstring(lua,"currentMiliseconds");
    lua_pushcfunction(lua,redis_math_currentMiliseconds);
    lua_settable(lua,-3);
 
    lua_setglobal(lua,"math");
    ...
}
 
int redis_math_currentMiliseconds(lua_State *L) {
    struct timeval now;
    gettimeofday(&now, NULL);
    lua_pushnumber(L, now.tv_sec*1000 + now.tv_usec/1000);
    return 1;
}
```

这个方案直接返回三元组（时间，分片ID，增长序列），当然Lua脚本是非常灵活的，可以自己随意修改。

```
时间：redis服务器上的毫秒数
分片ID：由传递进来的参数KEYS[1]%1024得到。
增长序列：由redis上"idgenerator_next_" 为前缀，接分片ID的Key用incrby命令得到。
```

例如，用户发一个文章，要生成一个文章ID，假定用户ID是14532，则

```
time <-- math.currentMiliseconds();
shardindId  <-- 14532 % 1024;     //即196
articleId <-- incrby idgenerator_next_196 1  //1是增长的步长
```

用lua脚本表示是：

```lua
local step = redis.call('GET', 'idgenerator_step');
local shardId = KEYS[1] % 1024;
local next = redis.call('INCRBY', 'idgenerator_next_' .. shardId, step);
return {math.currentMiliseconds(), shardId, next};
```

“idgenerator_step"这个key用来存放增长的步长。
客户端用eval执行上面的脚本，得到三元组之后，可以自由组合成64bit的全局ID。

**上面只是一个服务器，那么如何解决单点问题呢？**

上面的“idgenerator_step"的作用就体现出来了。

比如，要部署三台redis做为ID生成服务器，分别是A,B,C。那么在启动时设置redis-A下面的键值：

```
idgenerator_step = 3
idgenerator_next_1, idgenerator_next_2, idgenerator_next_3 ... idgenerator_next_1024 = 1
```

设置redis-B下面的键值：

```
idgenerator_step = 3
idgenerator_next_1, idgenerator_next_2, idgenerator_next_3 ... idgenerator_next_1024 = 2
```

设置redis-C下面的键值：

```
idgenerator_step = 3
idgenerator_next_1, idgenerator_next_2, idgenerator_next_3 ... idgenerator_next_1024 = 3
```

那么上面三台ID生成服务器之间就是完全独立的，而且平等关系的。任意一台服务器挂掉都不影响，客户端只要随机选择一台去用eval命令得到三元组即可。

我测试了下单台的redis服务器每秒可以生成3万个ID。那么部署三台ID服务器足以支持任何的应用了。

测试程序见这里：

https://gist.github.com/hengyunabc/9032295

缺点：

* 如果不熟悉lua脚本，可能定制自己的ID规则等比较麻烦。
* 注意机器时间不能设置为自动同步的，否则可能会因为时间同步，而导致ID重复了。

优点：

* 非常的快，而且可以线性部署。
* 可以随意定制自己的Lua脚本，生成各种业务的ID。

## 其它的东东

MongoDB的Objectid，这个实在是太长了要12个字节：

```
ObjectId is a 12-byte BSON type, constructed using:
 
a 4-byte value representing the seconds since the Unix epoch,
a 3-byte machine identifier,
a 2-byte process id, and
a 3-byte counter, starting with a random value.

```

## 总结

生成全局ID并不很难实现的东东，不过从各个网络的做法，及演进还是可以学到很多东东。有时候一些简单现成的组件就可以解决问题，只是缺少思路而已。

## 参考

* http://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/
* http://instagram-engineering.tumblr.com/post/10853187575/sharding-ids-at-instagram
* https://github.com/twitter/snowflake/
* http://docs.mongodb.org/manual/reference/object-id/
* http://www.redisdoc.com/en/latest/script/eval.html       redis脚本参考