---
layout: post
title: "Redis foundation - III"
date: "2022-06-27"
subtitle: "Redis basic knowledge - transaction & data persistence"
description: "Redis事务和持久化"
author: "Jamie Zhang"
image: "/img/background-05.jpg"
tags: 
    - Redis
    - NoSQL
categories: [ NoSQL ]
---

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

Redis Transaction Summary  
- All the operations in transaction will be executed sequentially and won't be interuptted by other clients.  
- All the operations queued won't take effect until 'exec' is executed  
- Redis cannot guarantee the atomicity.  
	
# Redis 持久化
## RDB - Redis Database
指定时间间隔内讲内存中的快照写到硬盘中。
具体步骤

- fork 一个进程作为原进程的子进程
- 写数据到临时区域/文件
- 覆盖临时文件到持久化dump.rdb文件，更好的保证数据完整性  
整个过程不影响主进程，缺点是最后一次持久化的数据可能会丢失  

> stop-writes-on-bgsave-error yes  停止接受写入命令如果rdb开启了并且最近一次后台save失败  
> rdbcompression yes 压缩string object using lzf when dump .rdb databases.  
> rdbchecksum yes CRC64算法进行数据校验,增加大约10%的性能消耗  
> save 900 1  - 900秒内有一个key变化就进行持久化  
	save 300 10 - 300秒内有10个key变化就吃法持久化

- 优点
	- 适合大规模数据恢复
	- 对数据完整性和一致性要求不高更合适使用
	- 节省磁盘空间
	-恢复速度快
- 缺点
	- fork操作导致了两倍的数据膨胀
	- 最后一次进行持久化后的数据可能会丢失，key的变化还没有来得及刷新到rdb文件.
### RDB备份
备份过程比较简单，根据save point time的设置，达到条件的改动就会被自动snapshot到dump.rdb, 备份即为备份dump.rdb文件，如果有故障发生则停掉redis，copy 备份的dump.rdb覆盖原dump.rdb，然后重新启动redis就回到了期望的版本。
	
```
# 获取dump.rdb的目录
127.0.0.1:6379> config get dir
1) "dir"
2) "/data"
```
## AOF - append only file
以日志的形式来记录每个写操作(增量),只许追加不允许修改，redis启动时会读取文件重新构建数据。
具体步骤  
- 客户端写请求被append到 AOF缓冲区  
- AOF缓冲区根据持久化策略将操作sync到磁盘的AOF文件中  
- AOF文件大小超过重写策略或手动重写时，会对AOF文件rewrite重新以压缩文件大小。

> AOF默认不开启,主要配置如下(redis.conf)：  
	- appendonly no   -- on/off switch  
	- appendfilename "appendonly.aof"  
	- appendfsync everysec  -- sync data from aof buffer to disk  
	- auto-aof-rewrite-percentage 100  
    - auto-aof-rewrite-min-size 64mb  -- minimal size for the AOF file to be rewritten even if the percentage increase is reached but it is still pretty small  
	
**important** if both RDB & AOF are enabled, redis restart will take "appendonly.aof" to restore data.  
	
异常恢复，如果aof文件损坏，可以尝试/usr/local/bin/redis-check-aof --fix appendonly.aof 进行恢复，然后重新重新启动redis。  

AOF同步策略：  
- appendfsync always   --  write log to appendonly.aof for each write operation immediately, performance is bad but good for data integrity.   
- appendfsync everysev   -- write logs on every seconds frequency, the last sec data might loss  
- appendfsync no  -- redis don't fsync data proactively, leave it to OS.  

### Rewrite 压缩
应对AOF文件越来越大，当大小超过阈值(文件大小和增加比例)，redis启动AOF内容压缩，保留恢复数据的最小指令集，可以强制压缩bgrewriteaof。

重写的流程：  
- fork出进程将文件重写(先写临时文件，然后在merge)，<span style="color:red">**redis 4.0**版本以后，是指把rdb的快照以二进制的形式附在新aof头部，作为已有的历史数据，替换掉原来的operation log</span>  

优点：   
- 备份机制更稳健，丢失数据的概率更低    
- 可读的日志文件通过AOF稳健减少处理错误操作  

缺点：   
- 比起rdb占用更多的磁盘空间  
- 恢复备份速度慢  
- 每次读写都同步的话，有一定的性能压力  


推荐两种持久化方式都启用，通常情况下用AOF文件内容来恢复数据，因为AOF文件保存的数据集要比RDB文件保存的数据集更加完整。  
RDB更适合数据库备份。