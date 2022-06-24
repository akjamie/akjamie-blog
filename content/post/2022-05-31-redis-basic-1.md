---
title: "Redis学习笔记"
date: 2022-05-30
subtitle: "Redis basic knowledge - 2"
description: "Redis基础知识回顾"
author: "Jamie Zhang"
image: "/img/background-05.jpg"
published: true
tags: 
    - Redis
    - NoSQL
categories: [ NoSQL ]
---
# Redis基础
##  Common知识
- Redis  有16个数据库，一般都使用0号库，使用select dbid 来切换db。
- Redis是单线程 + 多路io服用技术
- Redis vs memcache  
	- Redis ->  单线程 + 多路io复用 + 灵活支持多种数据类型
    - Memcached -> 多线程+锁 + 单一数据类型

- Key 操作 
	- keys * - list keys
	- expire key duration - set expire duration for key
	- exists key - verify if key exists
	- del key - delete key-value pair via key
	- unlink key - delete data in non-blocking way
	- ttl key - check remaining expiry duration

```
127.0.0.1:6379> dbsize
(integer) 7
127.0.0.1:6379> type test-set-1
set
127.0.0.1:6379> type test-zset-1
zset
127.0.0.1:6379> exists test-set-1
(integer) 1
127.0.0.1:6379> set test-string-1 "test" 
OK
127.0.0.1:6379> expire test-string-1 100
(integer) 1
127.0.0.1:6379> ttl test-string-1
(integer) 95
```
*  重要命令
	- select - select db
	- dbsize - check total key size in current db
	- flushdb - cleanup current db
	- flushall - cleanup all dbs
	
## 5大数据结构
* String 是redis最基本的类型，redis中字符串key和value最大可以存放512m
	- setex key ttl value
	- 存储方式为预分配冗余空间来减少内存的频繁分配
* List - 双向链表，一键多值
	- quickList, 元素少的时候是连续空间存储 - ziplist，多个ziplist
	- ziplist - 内存紧凑，访问效率高，缺点是更新效率低，并且数据量较大时，可能导致大量的内存复制
	- linkedlist - 节点修改的效率高，但是需要额外的内存开销，并且节点较多时，会产生大量的内存碎片
	- quicklist -  linkedlist 与 ziplist 的结合
* Set -自动排除数据重复
	- 数据架构师dict，基于hash表实现
	- Java 中HashSet的所有key指向同一个value，插入的元素即为key
* Hash
	- 数据结构，ziplist和hashtable，当field-value 长度 <=64,  默认使用ziplist，大于64位则自动使用hashtable
	
```
127.0.0.1:6379> object encoding test-hash-1
"ziplist"
127.0.0.1:6379> hset test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1 date 2022-05-30 event 'this is a testing event for redis hash testing' publisher 'system'
(integer) 3
127.0.0.1:6379> object encoding test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1
"ziplist"
127.0.0.1:6379> hset test-hash-2 test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1-test-hash-1 'test data encoding in redis'
(integer) 1
127.0.0.1:6379> object encoding test-hash-2
"hashtable"
```
* Zset 
	- 底层数据结构，类似于Map<String, Double>/TreeSet内部元素按照score进行排序
	- hash，关联元素value和权重，保证value的唯一性
	- skiplist，给value排序，根据score的范围获取元素
	
## 有序链表和跳跃表 
简单比较如下图，查找51，跳跃表的查找性能相较于链表效会更好。
<img src='/img/2022-05-26-redis-basic/linkedlist-vs-skiplist.png' style="height: 379px; margin-left: 0px;" />

## 新数据类型
* Bitmaps - 对位的操作
	- 其本身不是一种数据类型，实际上就是字符串
	- 可以对字符串的位进行操作
	- 提供了单独的操作命令集
	- 与set相比极大的节省内存空间，但存储内容有限
```
127.0.0.1:6379> setbit test:bitmaps:2 1 0
(integer) 0
127.0.0.1:6379> setbit test:bitmaps:2 2 0
(integer) 0
127.0.0.1:6379> setbit test:bitmaps:2 3 1
(integer) 0
127.0.0.1:6379> getbit test:bitmaps:1 1
(integer) 0
127.0.0.1:6379> getbit test:bitmaps:1 0
(integer) 1
```
* HyperLogLog - 做基数统计，耗费12kb内存，可以计算近2^64个不同元素的技术
	 - 相较于distinct，hash，set，bitmaps，更少的空间占用
	 - 用降低数据精确度来换存储空间
```
127.0.0.1:6379> pfadd hyperloglog:pageview:001 'jamie' 'james' 'edward'
(integer) 1
127.0.0.1:6379> pfadd hyperloglog:pageview:001 'jamie' 
(integer) 0
127.0.0.1:6379> pfadd hyperloglog:pageview:001 'john' 'edward'
(integer) 1
127.0.0.1:6379> pfcount hyperloglog:pageview:001
(integer) 4
127.0.0.1:6379> pfadd hyperloglog:pageview:002 'john' 'edward'
(integer) 1
127.0.0.1:6379> pfmerge hyperloglog:pageview:summ hyperloglog:pageview:001 hyperloglog:pageview:002
OK
127.0.0.1:6379> pfcount hyperloglog:pageview:summ
(integer) 4
127.0.0.1:6379> get hyperloglog:pageview:summ
"HYLL\x01\x00\x00\x00\x04\x00\x00\x00\x00\x00\x00\x00M\xca\x84^\xa1\x80I\x86\x84E]\x84D\xa9"
```

* Geospatial - 地理经纬度
	- geohash保存地理位置的坐标
	- geohash计算的字符串，可以反向解码出原来的经纬度
	- 有序集合（zset）保存地理位置数据, member + geohash
```
127.0.0.1:6379> geoadd china:city:xian 108.84067296981812 34.23806027872841 lvdi
(integer) 1
127.0.0.1:6379> geoadd china:city:xian 108.88047695159912 34.22972243813365 gaoxinhospital
(integer) 1
127.0.0.1:6379> geoadd china:city:xian 108.84039402008057 34.2350800390342 zhenguanfu
(integer) 1
127.0.0.1:6379> geoadd china:city:xian 108.85704517364502 34.245226613720426 mincheng
(integer) 1
127.0.0.1:6379> geoadd china:city:xian 108.8686752319336 34.22915472534989 waishi
(integer) 1
127.0.0.1:6379> geopos china:city:xian lvdi
1) 1) "108.84067565202713013"
   2) "34.23805909783406065"
127.0.0.1:6379> geodist china:city:xian lvdi zhenguanfu km
"0.3323"
127.0.0.1:6379> geodist china:city:xian lvdi gaoxinhospital km
"3.7758"
127.0.0.1:6379> georadius china:city:xian 108.84067565202713013 34.23805909783406065 3 km
1) "zhenguanfu"
2) "lvdi"
3) "waishi"
4) "mincheng"
127.0.0.1:6379> GEORADIUSBYMEMBER china:city:xian lvdi 3 km
1) "zhenguanfu"
2) "lvdi"
3) "waishi"
4) "mincheng"
127.0.0.1:6379> GEOHASH china:city:xian lvdi
1) "wqj6u8z75s0"
127.0.0.1:6379> OBJECT ENCODING china:city:xian
"ziplist"
```


# Redis 重要配置
## redis.conf
* timeout - Close the connection after a client is idle for N seconds
* pidfile - /var/run/redis_6379.pid
* loglevel - debug/verbose/notice/warning
* databases - the number of databases
* requirepass - setting password for default user
```
root@d5ff6116c0ab:/# redis-cli
127.0.0.1:6379> config get requirepass
1) "requirepass"
2) ""
127.0.0.1:6379> config set requirepass '123456'
OK
127.0.0.1:6379> config get requirepass
1) "requirepass"
2) "123456"
127.0.0.1:6379> auth '123456'
OK
```
* maxclients - the max number of connected clients at the same time
* maxmemory - memory usage limit, 强烈推荐设置，否则可能会内存耗尽而宕机
* maxmemory-policy - how Redis will select what to remove when maxmemory is reached
   - volatile-lru - Least Recently Used, 只对设置了ttl的keys
   - allkeys-lru - all keys
   - volatile-lfu - Least Frequently Used, 只对设置了ttl的keys
   - allkeys-lfu
   - volatile-random - 只对设置了ttl的keys
   - allkeys-random
   - volatile-ttl - 只对设置了ttl的keys
   - noeviction - 不移除，写入时返回错误
*  maxmemory-samples - LRU, LFU and minimal TTL algorithms are not precise algorithms but approximated，by default redis Redis will check five keys and pick the one that was used least recently


# 发布与订阅
<img src='/img/2022-05-26-redis-basic/pub-sub-01.jpg' style="height: 294px; margin-left: 0px;" />

# 示例场景
* 短信验证码

> 主要思路：  
1>  随机生成6位数字码  
2> 存储验证码到redis，且设置ttl(如2mins), sample key pattern: <app-name/system-name>:<phone-number>:<verificaiton-code>, e.g. 'user-service:13992590000:678325'  
3> 其他限制，如每天每个手机号可以发送的验证码的次数，设置单独的key来管理发送次数，sample key pattern：<app-name/system-name>:<phone-number>:count， 如果次数超过limit则不生成验证码返回exception  
4> 调用短信服务商接口发送短信  
[redis-test-sample](https://github.com/cloud-poc/redis-test)

* 静态数据缓存加速前端访问或者缓冲db的i/o压力
	
# Redis 事务
## 事务概念和基本操作
其为命令提供一个单独的隔离环境，事务中所有的命令都会序列化，按顺序地执行，执行过程中不会被其他客户端发送过来的命令请求打断。
与dbms事物不同，redis事务的作用就是串联多个命令防止被其他命令插队

- discard sample
```
127.0.0.1:6379[1]> keys *
(empty array)
127.0.0.1:6379[1]> multi
OK
127.0.0.1:6379[1]> set key-1 test-1
QUEUED
127.0.0.1:6379[1]> mset key-2 test-2 key-3 test-3
QUEUED
127.0.0.1:6379[1]> discard
OK
127.0.0.1:6379[1]> keys *
(empty array)
```
- exec succ

```
127.0.0.1:6379[1]> keys *
(empty array)
127.0.0.1:6379[1]> multi
OK
127.0.0.1:6379[1]> mset key-2 test-2 key-3 test-3
QUEUED
127.0.0.1:6379[1]> exec
1) OK
127.0.0.1:6379[1]> keys *
1) "key-2"
2) "key-3"
127.0.0.1:6379[1]>
```
- error in middle of exec

```
127.0.0.1:6379[1]> multi
OK
127.0.0.1:6379[1]> hget hkey-1 color
QUEUED
127.0.0.1:6379[1]> incr key-2
QUEUED
127.0.0.1:6379[1]> hset hkey-1 color blue
QUEUED
127.0.0.1:6379[1]> exec
1) (nil)
2) (error) ERR value is not an integer or out of range
3) (integer) 1
127.0.0.1:6379[1]> hget hkey-1 color
"blue"
```

## Redis 乐观锁
watch key - monitor one or more key
	
- sample
```
127.0.0.1:6379[1]> watch key-1 lkey-1
OK
127.0.0.1:6379[1]> incrby key-1 100
(integer) 300
127.0.0.1:6379[1]> multi
OK
127.0.0.1:6379[1]> incrby key-1 100
QUEUED
127.0.0.1:6379[1]> lpush lkey-1 400
QUEUED
127.0.0.1:6379[1]> exec
(nil)
127.0.0.1:6379[1]>
```
ps: the keys 'watched' has been changed by other client, hence the transaction executed failed.

## Redis Transaction Summary
- All the operations in transaction will be executed sequentially and won't be interuptted by other clients.
- All the operations queued won't take effect until 'exec' is executed
- Redis cannot guarantee the atomicity.
	
# Redis 持久化
## RDB
指定时间间隔内讲内存中的快照写到硬盘中。
具体步骤

- fork 一个进程作为原进程的子进程
- 写数据到临时区域/文件
- 覆盖临时文件到持久化dump.rdb文件，更好的保证数据完整性  
整个过程不影响主进程，缺点是最后一次持久化的数据可能会丢失  

> stop-writes-on-bgsave-error yes  停止接受写入命令如果rdb开启了并且最近一次后台save失败
> rdbcompression yes 压缩string object using lzf when dump .rdb databases.
> rdbchecksum yes CRC64算法进行数据校验
> save 900 1  - 900秒内有一个key变化就进行持久化
	save 300 10 - 300秒内有10个key变化就吃法持久化

- 优点
	- 适合大规模数据恢复
	- 对数据完整性和一致性要求不高更合适使用
	- 节省磁盘空间
	- 回复速度快
- 缺点
	- fork操作导致了两倍的数据膨胀
	- 最后一次进行持久化后的数据可能会丢失，key的变化还没有来得及刷新到rdb文件
	
	
	
# Referred Articles
https://www.cnblogs.com/LBSer/p/3310455.html   
http://redisbook.com/preview/object/sorted_set.html





	