---
title: "Redis学习笔记"
date: 2022-06-28
subtitle: "Redis basic knowledge - Master/slave replication"
description: "Redis主从复制"
author: "Jamie Zhang"
image: "/img/background-05.jpg"
published: true
tags: ["Redis", "NoSQL", "K8S"]
categories: [ NoSQL ]
---
# 主从复制
## 基本概念
主机数据更新后根据配置和策略，自动同步到备份机的master/slave机制，其中master以写为主，而slave主要以读为主。    
该机制使   
1》读写分离，性能获得提升  
2》容灾快速恢复

## 环境搭建
快速demo搭建一主两从的主从复制集群，准备3份配置文件：  
- redis-6379.conf  
- redis-6380.conf   
- redis-6381.conf    

```
# -----------------------------------------------------------------------
# redis-6379.conf ==> master node
# include original redis config - full version
include /Users/jamie/Documents/work-benches/redis/redis-6.0.6/redis.conf

#pidfile path
pidfile /var/run/redis-6379.pid

#redis listening port
port 6379

#rdb filename
dbfilename dump-6379.rdb

#appendonly filename
appendfilename appendonly-6379.aof

daemonize yes
# -----------------------------------------------------------------------

# redis-6380.conf ==> slave node
# include original redis config - full version
include /Users/jamie/Documents/work-benches/redis/redis-6.0.6/redis.conf

#pidfile path
pidfile /var/run/redis-6380.pid

#redis listening port
port 6380

#rdb filename
dbfilename dump-6380.rdb

#appendonly filename
appendfilename appendonly-6380.aof

daemonize yes

# slave of master node
replicaof 127.0.0.1 6379
# -----------------------------------------------------------------------

# redis-6381.conf ==> slave node
# include original redis config - full version
include /Users/jamie/Documents/work-benches/redis/redis-6.0.6/redis.conf

#pidfile path
pidfile /var/run/redis-6381.pid

#redis listening port
port 6381

#rdb filename
dbfilename dump-6381.rdb

#appendonly filename
appendfilename appendonly-6381.aof

daemonize yes

replicaof 127.0.0.1 6379
# -----------------------------------------------------------------------
```

启动redis实例  'nohup redis-server ./config/redis-<port>.conf', 当3个实例启动完成后, 在master 上检查replication的配置,看到如下信息则主从配置成功 
<img src='/img/2022-06-28-redis-master-slave-replication/replica-01.png' style="height: 428px;margin-left: 0px;"/>

3中加入到复制集的方式  
- 依赖于config文件中的__replicaof {masterhost} {masterport}__命令把节点加入到集群  
- 通过 redis-server --slaveof {masterhost} {masterport}进行添加  
- 在slave节点上运行redis 命令 slaveof {masterhost} {masterport}进行添加

## 注意问题或重点

1. 从服务器挂掉之后重新启动，已经不再是从模式(只有在conf中没有replicaof的情况下)，需要重新手动添加到集群，主服务器重新集群模式仍然不变。  
2. 主从复制原理
	- salve连接上master之后，salve向master请求数据同步sync
	- master收到同步数据请求后，进行数据持久化到rdb文件，发送rdb文件给slave
	- 每次master写操作，主动同步到slave
3. slave 上可以添加slave，形成树形结构
4. 主从切换，当master挂掉，slave可以升级为master
	- 手动进行切换 --〉 slaveof no one
	- 哨兵模式 sentinel
	
## 哨兵模式 sentinel
哨兵模式是一种特殊的模式，首先Redis提供了哨兵的命令，哨兵是一个独立的进程，作为进程，它会独立运行。其原理是**哨兵通过发送命令，等待Redis服务器响应，从而监控运行的多个Redis实例**.

### 启动哨兵
copy redis安装目录下的sentinel.conf配置修改如下：
```
# protection mode
protected-mode no
# monitor master node
sentinel monitor mymaster 127.0.0.1 6379 1	
```

启动哨兵 - **redis-sentinel ./config/sentinel.conf**, 如果启动正常可以在日志中看到master和slave节点的信息。
<img src='/img/2022-06-28-redis-master-slave-replication/sentinel-01.png' style="weight: 1450px;height: 132px;margin-left: 0px;"/>

### 验证failover

#### 验证前server信息  
	
| 节点 | 端口 | 角色 |
|--- |--- | --- |
| 127.0.0.1|6379|master|
| 127.0.0.1|6380|slave|
| 127.0.0.1|6381|slave|
| 127.0.0.1|26379|<span style='color:red'>sentinel</span>|

输出replication详情如下：
```
Jamies-MacBook-Pro:workspace-20190620 jamie$ redis-cli -p 6379
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online,offset=43115,lag=0
slave1:ip=127.0.0.1,port=6381,state=online,offset=43115,lag=1
master_replid:e2dd09590dc64383a7a97d439ca70ae61a8f13b0
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:43115
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:43115
```
#### failover后的server 信息
手动shutdown 6379，查看sentinel日志获取新的主节点信息，并输出新主节点上的replication info.
<img src='/img/2022-06-28-redis-master-slave-replication/sentinel-02.png' style="weight: 1852px;height: 654px;margin-left: 0px;"/>
由上图所示，新的master 为6380.
```
Jamies-MacBook-Pro:workspace-20190620 jamie$ redis-cli -p 6380
127.0.0.1:6380> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6381,state=online,offset=90136,lag=1
```
综上所述，新的server info 为

| 节点 | 端口 | 角色 |
|--- |--- | --- |
| 127.0.0.1|6379|slave after restarted|
| 127.0.0.1|6380|master|
| 127.0.0.1|6381|slave|
| 127.0.0.1|26379|<span style='color:red'>sentinel</span>|

### Spring + Lettuce 测试failover
配置如下：
```
# application.yml ==>
spring:  
  redis:  
    host: localhost  
    port: 6380  
    lettuce:  
      pool:  
        max-active: 3  
        max-idle: 2  
        min-idle: 1  
        max-wait: 10000  
    sentinel:  
      master: mymaster  
    client-type: lettuce
```

Test case - 每间隔1分钟向redis写一条数据，在过程shutdown 6380，等待sentinel发现并切换主节点然后应用恢复连接并继续写入数据。
```
@SpringBootTest  
@Slf4j  
public class RedisWriteOperationWithFailover {  
    @Autowired  
    RedisTemplate<Object, Object> redisTemplate;  
  
    @Test  
    public void test() {  
        final String testKeyPrefix = "test:sentinel:sequence:";  
        IntStream.range(0, 20).forEach(i -> {  
            String key = testKeyPrefix + i;  
            log.info("write record to redis, key: {}, value:{}", key, i);  
            redisTemplate.opsForValue().set(key, i, Duration.ofMinutes(1));  
            try {  
                TimeUnit.MINUTES.sleep(1);  
            } catch (InterruptedException e) {  
                throw new RuntimeException(e);  
            }  
        });  
    }  
}
```
测试结果如图
<img src='/img/2022-06-28-redis-master-slave-replication/sentinel-03.png' style="height: 727px;margin-left: 0px;"/>
从上图中得知：  
- 2022-07-07 23:34:58发现master down机，然后尝试重新连接一直fail。  
- sentinel在30秒后进行投票选举新的master 并切换到新master 6379。  
- 应用自动连接到新的master回复数据写入。  

# Refference Articles
https://www.jianshu.com/p/f2cb77e7a402 