---
title: "Redis advance III"
date: "2022-07-09"
subtitle: "Redis basic knowledge - Redis cluster - Part I"
description: "Redis 集群"
author: "Jamie Zhang"
image: "/img/background-05.jpg"
tags: ["Redis", "NoSQL"]
categories: [ NoSQL ]
---

# What's Redis Cluster
Redis Cluster is a distributed implementation of Redis with the following goals in order of importance in the design:

-   High performance and linear scalability up to 1000 nodes. There are no proxies, asynchronous replication is used, and no merge operations are performed on values.  
-  在多达 1000 个节点的时候仍能保持高性能及线性的可扩展性。没有代理，使用异步复制，并且不对值执行合并操作  
-   Acceptable degree of write safety: the system tries (in a best-effort way) to retain all the writes originating from clients connected with the majority of the master nodes. Usually there are small windows where acknowledged writes can be lost. Windows to lose acknowledged writes are larger when clients are in a minority partition.  
-   可接受的写入安全：与多数派节点相连的客户端所做的写入操作，系统尝试全部都保存下来（以最大努力的方式）。不过极小概率下容忍小部分写入会丢失  
-   Availability: Redis Cluster is able to survive partitions where the majority of the master nodes are reachable and there is at least one reachable replica for every master node that is no longer reachable. Moreover using _replicas migration_, masters no longer replicated by any replica will receive one from a master which is covered by multiple replicas.  
-   可用性：分区情况下在多数派这边，当绝大多数的主节点是可达的，并且对于每一个不可达的主节点都至少有一个它的从节点可达的情况下，Redis Cluster 仍能提供服务  

# Local redis cluster
Exploration starts from setting up a local redis cluster, the simulated nodes listed as below:  

| Node| Port| Role | Remark|
|--- | --- | --- |---|
| 192.168.0.110 | 6379 | master |master node with 1 slave|
| 192.168.0.110 | 6380 | master |master node with 1 slave|
| 192.168.0.110 | 6381 | master |master node with 1 slave| 
| 192.168.0.110 | 6389 | slave |slave of 6381|
| 192.168.0.110 | 6390 | slave |slave of 6379|
| 192.168.0.110 | 6391 | slave |slave of 6380|

## Redis config
Config file template:
```
# include original redis config - full version
include /Users/jamie/Documents/work-benches/redis/redis-6.0.6/redis.conf

#pidfile path
pidfile /var/run/redis-6379.pid

#redis listening port
port {port}

#rdb filename
dbfilename dump-{port}.rdb

#open the appendonly
appendonly yes

#appendonly filename
appendfilename appendonly-{port}.aof

daemonize yes

bind 0.0.0.0

#enable redis cluster mode
cluster-enabled yes

# config file auto mantained by redis
cluster-config-file node-{port}.conf

# timeout setting for cluster, if get timeout, then start to switch master/slave
cluster-node-timeout 15000
```
## Steps to startup
1. Use __redis-server ./config/redis-{port}.conf__ to start each instance/node.
2. Merge the started redis instance/nodes into one cluster.
```
redis-cli --cluster create --cluster-replicas 1 192.168.0.110:6379 192.168.0.110:6380 192.168.0.110:6381 192.168.0.110:6389 192.168.0.110:6390 192.168.0.110:6391

>> 
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.0.110:6390 to 192.168.0.110:6379
Adding replica 192.168.0.110:6391 to 192.168.0.110:6380
Adding replica 192.168.0.110:6389 to 192.168.0.110:6381
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 58b3c8fcc4546f122e42f546bd015c729bd4a531 192.168.0.110:6379
   slots:[0-5460] (5461 slots) master
M: 68913fbd6577582d8aba73be69f7c38430c9cb37 192.168.0.110:6380
   slots:[5461-10922] (5462 slots) master
M: 44312cf65f32aaeae033a01094c41bea947b7946 192.168.0.110:6381
   slots:[10923-16383] (5461 slots) master
S: 3a506595ed87964b42ef135d8862acef0613626e 192.168.0.110:6389
   replicates 68913fbd6577582d8aba73be69f7c38430c9cb37
S: 4d2ade8214274a1653d2ba68f0ab12f0e61502d5 192.168.0.110:6390
   replicates 44312cf65f32aaeae033a01094c41bea947b7946
S: c1682ccc3e5a86dafc78fe7a553a505718f4e9ba 192.168.0.110:6391
   replicates 58b3c8fcc4546f122e42f546bd015c729bd4a531
```

## Key points summary

> 1. <span style="color: red; font-weight: bold">cluster-replicas 1 </span> ==>   
specify 1 master 1 slave mode,should guarantee the master and slave node are not on same physical machine.

> 2. <span style="color: red; font-weight: bold">Slot</span> ==>   
when insert key-value pair to redis, redis will use algorithm [CRC16(KEY)] to get a slot number, then map to redis master node according to their slot ranges of each master node.

> 3. Cross Slot data insertion will be rejected in cluster mode ==>  
There is also an approach to solve the CrossSlot error at application level, use hashtags ({…}) to force the keys into the same hash slot.  
```
192.168.0.110:6389> mset name Jamie age 18
(error) CROSSSLOT Keys in request don't hash to the same slot

192.168.0.110:6389> mset {user-01}name Jamie age{user-01} 18 height:{user-01} 175 {user-01}:weight 70
-> Redirected to slot [1116] located at 192.168.0.110:6391
OK
192.168.0.110:6391> cluster keyslot {user-01}name
(integer) 1116
192.168.0.110:6391> cluster keyslot {user-01}age
(integer) 1116
192.168.0.110:6391> cluster keyslot {user-01}-age
(integer) 1116
192.168.0.110:6391> cluster keyslot {user-01}:age
(integer) 1116
```

Sample Hash Slot calculation:

|Key	|Hashing Pseudocode	|Hash Slot|
|--- |--- | --- |
|user-profile:1234	  |  CRC16(‘user-profile:1234’) mod 16384	|15990|
|user-session:1234	  |  CRC16(‘user-session:1234’) mod 16384	|2963 |
|user-profile:5678	  |  CRC16(‘user-profile:5678’) mod 16384	|9487 |
|user-session:5678	  |  CRC16(‘user-session:5678’) mod 16384	|4330 |
|user-profile:{1234}  |	 CRC16(‘1234’) mod 16384	            |6025 |
|user-session:{1234}  |	 CRC16(‘1234’) mod 16384	            |6025 |
|user-profile:{5678}  |	 CRC16(‘5678’) mod 16384	            |3312 |
|user-profile:{5678}  |	 CRC16(‘5678’) mod 16384	            |3312 | 

Check assigned hash slot by a given key on cluster:   
```
192.168.0.110:6389> cluster keyslot test

(integer) 6918
```


Other useful commands  
- cluster countkeysinslot {slot number}  
- cluster getkeysinslot {slot number} {count}

> these two commands can only see the own slots for each master node.

# Auto Failover
When a master fails or is found to be unreachable by the majority of the cluster as determined by the nodes communication via the gossip port, the remaining masters hold a vote and elect one of the failing masters’ slaves to take its place.

Do a demo in local based on the cluster already setup in previous steps, will shutdown 6391 and check the cluster status, the expected behavior is <span style='background-color: yellow'>the 6379 will be elected as master node to replace the 6391, and the 6391 will be worked as slave node after restarted.</span>  

Firstly, capture exist redis node status for later comparison.  
```
# run cluster nodes on any cluster nodes
c1682ccc3e5a86dafc78fe7a553a505718f4e9ba 192.168.0.110:6391@16391 master - 0 1657641720841 7 connected 0-5460

3a506595ed87964b42ef135d8862acef0613626e 192.168.0.110:6389@16389 master - 0 1657641719834 15 connected 5461-10922

58b3c8fcc4546f122e42f546bd015c729bd4a531 192.168.0.110:6379@16379 myself,slave c1682ccc3e5a86dafc78fe7a553a505718f4e9ba 0 1657641719000 7 connected

68913fbd6577582d8aba73be69f7c38430c9cb37 192.168.0.110:6380@16380 slave 3a506595ed87964b42ef135d8862acef0613626e 0 1657641718825 15 connected

4d2ade8214274a1653d2ba68f0ab12f0e61502d5 192.168.0.110:6390@16390 master - 0 1657641720000 14 connected 10923-16383

44312cf65f32aaeae033a01094c41bea947b7946 192.168.0.110:6381@16381 slave 4d2ade8214274a1653d2ba68f0ab12f0e61502d5 0 1657641717000 14 connected
```

Secondly, shutdown the master node 6391, the 6379(slave of 6391) will be new master.
```
127.0.0.1:6389> cluster nodes

3a506595ed87964b42ef135d8862acef0613626e 192.168.0.110:6389@16389 myself,master - 0 1657641905000 15 connected 5461-10922

c1682ccc3e5a86dafc78fe7a553a505718f4e9ba 192.168.0.110:6391@16391 master,fail - 1657641873514 1657641871500 7 disconnected

44312cf65f32aaeae033a01094c41bea947b7946 192.168.0.110:6381@16381 slave 4d2ade8214274a1653d2ba68f0ab12f0e61502d5 0 1657641903000 14 connected

58b3c8fcc4546f122e42f546bd015c729bd4a531 192.168.0.110:6379@16379 master - 0 1657641904841 17 connected 0-5460

68913fbd6577582d8aba73be69f7c38430c9cb37 192.168.0.110:6380@16380 slave 3a506595ed87964b42ef135d8862acef0613626e 0 1657641905851 15 connected

4d2ade8214274a1653d2ba68f0ab12f0e61502d5 192.168.0.110:6390@16390 master - 0 1657641903831 14 connected 10923-16383
```

Thirdly, restart 6391, the 6391 will be slave of 6379.
```
Jamies-MacBook-Pro:cluster jamie$ redis-server ./config/redis-6391.conf

Jamies-MacBook-Pro:cluster jamie$ redis-cli -p 6389

127.0.0.1:6389> cluster nodes

3a506595ed87964b42ef135d8862acef0613626e 192.168.0.110:6389@16389 myself,master - 0 1657642036000 15 connected 5461-10922

c1682ccc3e5a86dafc78fe7a553a505718f4e9ba 192.168.0.110:6391@16391 slave 58b3c8fcc4546f122e42f546bd015c729bd4a531 0 1657642037186 17 connected

44312cf65f32aaeae033a01094c41bea947b7946 192.168.0.110:6381@16381 slave 4d2ade8214274a1653d2ba68f0ab12f0e61502d5 0 1657642038196 14 connected

58b3c8fcc4546f122e42f546bd015c729bd4a531 192.168.0.110:6379@16379 master - 0 1657642038000 17 connected 0-5460

68913fbd6577582d8aba73be69f7c38430c9cb37 192.168.0.110:6380@16380 slave 3a506595ed87964b42ef135d8862acef0613626e 0 1657642039201 15 connected

4d2ade8214274a1653d2ba68f0ab12f0e61502d5 192.168.0.110:6390@16390 master - 0 1657642036000 14 connected 10923-16383
```

What if both master and it's slaves are down, then depends on the config value of <span style='color:red'> cluster-require-full-coverage </span>, if cluster-require-full-coverage=yes, then whole custer cannot work; if no, then only the hash slots of the unavailable master cannot work.

# Pros and Cons
pros:   
- High Performance, promises the same level of performance as standalone Redis deployments.  
- High Availability, supports the standard Redis master-replica configuration  
- Horizontal & Vertical Scalability  

cons:   
- Requires Client to dynamically detect the slots mapping, limited redis client can be used.
- Data sync is processed asynchronously, cannot guarantee the data Strong Consistence.
- Slave node only can only work as data backup, cannot balance cluster read/write pressure.
- Limited multi-keys operations(mset, mget, sunion,etc.) are supported as multi-key operations are supported only when all the keys in a single operation belongs to the same slot.    
- Lua is not supported


# Reference docs
https://redis.io/docs/reference/cluster-spec/
