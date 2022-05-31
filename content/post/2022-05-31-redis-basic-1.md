---
title: "NoSQL DB & Distributed cache - Redis"
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
# Redis基础回顾
* Redis  有16个数据库，一般都使用0号库，使用select dbid 来切换db。
* Redis是单线程 + 多路io服用技术
* Redis vs memcache  
	
	 | Product| 架构模型 | 数据类型支持|  
	 | :--------- | :------------- | :---------- |  
	 | Redis | 单线程 + 多路io复用|  多种数据类型|  
	 | Memcached | 多线程+锁|  单一数据类型 |
	
* Key 操作 
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
* 有序链表和跳跃表 
	<img src='/img/2022-05-26-redis-basic/linkedlist-vs-skiplist.png style="height: 379px; margin-left: 0px;"/>

	