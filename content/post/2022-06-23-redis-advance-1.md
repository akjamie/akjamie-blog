---
title: "NoSQL DB & Distributed cache - Redis"
date: 2022-06-23
subtitle: "Redis advanced knowledge"
description: "Redis进阶知识总结"
author: "Jamie Zhang"
image: "/img/background-05.jpg"
published: true
tags: 
    - Redis
    - NoSQL
categories: [ NoSQL ]
---
# bitmap
在前面的基础回顾中提到了bitmap这种数据结构，主要用来应对string的位操作，简单高效
适用场景：单状态统计，如每天系统用户登录情况统计，github commit
其常用命令有 setbit, getbit, bitcount, bitop.
```
127.0.0.1:6379> setbit test:bitmaps:2 0 1
(integer) 0
127.0.0.1:6379> bitcount test:bitmaps:2
(integer) 1
127.0.0.1:6379> setbit test:bitmaps:2 0 2   --> the value on each bit is only 0 and 1
(error) ERR bit is not an integer or out of range
127.0.0.1:6379> setbit test:bitmaps:2 1 1
(integer) 0
127.0.0.1:6379> setbit test:bitmaps:2 3 1
(integer) 0
127.0.0.1:6379> bitcount test:bitmaps:2    --> 1101
(integer) 3
127.0.0.1:6379> bitop not notValue test:bitmaps:2 --> 0010
(integer) 1
127.0.0.1:6379> getbit notValue 0
(integer) 0
127.0.0.1:6379> getbit notValue 1
(integer) 0
127.0.0.1:6379> getbit notValue 2
(integer) 1
```



# Reference
https://zhuanlan.zhihu.com/p/480386998
