---
title: "Redis学习笔记"
date: 2019-08-04
subtitle: "Redis basic knowledge"
excerpt: ""
description: "redis基础知识"
author: "Jamie Zhang"
image: "/img/background-05.jpg"
published: true
tags: 
    - Redis
    - NoSQL
categories: [ NoSQL ]
---
redis是一个key-value存储系统，它支持存储的value类型相对更多，包括string(字符串)、list(链表)、set(集合)、zset(sorted set --有序集合)和hash（哈希类型）。这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。在此基础上，redis支持各种不同方式的排序。与memcached一样，为了保证效率，数据都是缓存在内存中。区别的是redis会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件，并且在此基础上实现了master-slave(主从)同步。
# Redis的优势与缺点:
## 优势
    • 性能极高，redis读的速度110000次/秒，写的速度81000次/秒
    • 丰富的数据类型，redis支持的类型String, List, Hash, Set 和Ordered Set
    • 原子性，redis的所有操作都是原子性的，多个操作也支持事务
    • 丰富的特性，如pub/sub,key过期等
## 缺点
    • 耗内存，内存占用过高
    • 持久化，Redis直接将数据存储到内存中，要将数据保存到磁盘上，Redis可以使用两种方式实现持久化， 
       定时快照(snapshot)：每隔一段时间将整个数据库写到磁盘上，每次军写全部数据，代价非常高。
       基于语句追加(aof): 只追踪变化的数据，但是追加的log可能过大，同时所有的操作均重新执行一边，回复速度慢。
# Redis.conf 配置说明: 
    • Daemonize   no|yes   是否启用守护进程
    • Pidfile  /var/run/redis.pid
    • Port  6379    端口  redis监听端口
    • Database 16  设置数据库的数目。
    • loglevel notice 
      # debug (适用于开发或测试阶段)
      # verbose (many rarely useful info, but not a mess like the debug level)
      # notice (适用于生产环境)
      # warning (仅仅一些重要的消息被记录)
    • 快照 - 存db到磁盘  - 根据给定的时间间隔和写入次数将数据保存到磁盘
    • save 900 1
    • save 300 10
    • save 60 10000
    • stop-writes-on-bgsave-error yes 如果 redis 最后一次的后台保存失败，redis 将停止接受写操作
    • dbfilename dump.rdb   设置 dump 的文件位置
    • …

# Redis 安装，连接与关机
## Docker 方式安装
通过docker compose命令进行安装，docker-compose  file请参考 [sample](https://github.com/cloud-poc/devops-config-service-repo/blob/master/docker-compose-redis.yml)
## Redis连接
```
redis-cli  -h <host> -p 6379 -a <password>
```
## Redis 关机
```
#强制结束进程，数据可能会丢失
ps -ef | grep -i redis
Kill -9 PID

#正常关机，会保存数据
redis-cli shutdown
```

# 数据类型
## String
String是redis最基本的数据类型，一个key对应一个value，且string是二进制安全的，一个键能存储512m数据  
    ○ 二进制安全是指，在数据传输过程中，保证二进制数据安全不被篡改，破译等，如果被攻击能检测出来  
    ![](/img/2019-08-04-redis-primary-knowledge/binary-safe.png)
    ○ 编码和解码发生在客户端，执行效率高，不需要频繁编解码，不会出现乱码
### 常用命令:
    • SET key value  #设置给定key的值，如果key已经存在就覆盖旧值，且无视数据类型
    • SETNX key value  #只有在key不存在是设置key的值，(SET if Not eXists)
    • MSET key value [key value]  #同时设置一个或者多个key-value
    • GET key  #获取指定key的值，如果可以不存在返回nil(值为string类型，否则会报错)
    • GETRANGE key start end  #获取key对应的字符串的字串，截取由偏移量start 和end决定
    • MGET key1 [key2]  #获取一个或者多个key的值
    • GETSET key value  #设置指定的key的值，并返回key的旧值，当key不存在时返回nil
    • STRLEN key  #返回key对应字符串值的长度
    • DEL key  #删除指定key
    • INCR key  #将可以对应的数值自增,key不存在则先初始化为0，然后再执行incr操作，并返回操作后的结果
    • INCRBY key 增量值  #将key对应的数值加上指定的增量值，key不存在的处理同incr，并返回操作后的结果
    • DECR key 和 DECRBY key 减量值  #与incr/incrby相反
    • APPEND key name  #追加value到key对应的值的末尾，如果不存在key，则等同于 set key name
## Hash, 类似于hashmap的存储结构
 将一个对象存储为hash类型，较于每个字段都存储成string类型更能节省内存。新建一个hash对象时开始是用zipmap(又称为small hash)来存储的。这个zipmap其实并不是hash table，但是zipmap相比正常的hash实现可以节省不少hash本身需要的一些元数据存储开销。尽管zipmap的添加，删除，查找都是O(n)，但是由于一般对象的field数量都不太多。所以使用zipmap也是很快的,也就是说添加删除平均还是O(1)。如果field或者value的大小超出一定限制后，Redis会在内部自动将zipmap替换成正常的hash实现。
### 底层存储方式配置参数:
```
hash-max-zipmap-entries 512 #配置字段最多512个
hash-max-zipmap-value 64 #配置value最大为64字节
```

### 对象存储，string vs hash
  ○ 将用户id作为查找key，把其他信息封装成一个对象以序列化的方式存储(含json方式)，增加了序列化/反序列化的开销，同时，如果需要修改其中的一个属性值的时候，需要把整个对象取回，并且修改操作需要进行并发保护，引入更多复杂问题  
  ○ 把该对象拆分为多个key/value pair， 用id:fieldName 来做为唯一标识该属性，该种方式省区了序列化和并发问题，但是id为重复存储，如果大量存在这种数据，内存浪费会比较可观  
  ○ Hash内部存储的value为一个HashMap，并提供了直接存取这个Map成员的接口
### 常用命令
    • HSET key field value  #为指定的key，设置field/value 对
    • HMSET key field value [field 1 value1]  #将一对儿或者多对儿 field/value设置到key中
    • HGET key field  #获取hash 中的值，根据field 获取到value
    • HMGET key field [field1]  # 获取key 指定fields的值
    • HGETALL key  #返回key对应的所有field的值
    • HKEYS key  #获取key对应的所有字段名
    • HVALS key  #获取key对应的所有字段的值
    • HLEN key  #获取key对应的字段数量
    • HDEL key field [field1]  #删除key下面的一个或者多个字段
    • HSETNX key field value  #只有在 field不存在时，设置字段值
    • HINCRBY key field increment  #整数字段值加上增量并返回
    • HINCRBYFLOAT key field increment  #浮点数字段值加上增量并返回
    • HEXISTS key field  #指定字段是否存在
## List
  类似于linkedList，双向链表结构
### 常用命令
    • LPUSH key value1 [value2]  #将一个或者多个值插入到列表头部
    • RPUSH key value1 [value2]  #从右侧添加一个或者多个值
    • LPUSHX key value  #将一个值插入到已经存在的列表头部，如果列表不存在，则操作无效
    • RPUSHX key value  #将一个值插入到已经存在的列表尾部，如果列表不存在，则操作无效
    • LLEN key  #获取列表长度
    • LINDEX key index  #通过索引获取列表中的元素
    • LRANGE key start stop  #获取列表指定范围内的元素
    • LPOP key  #移出并获取列表中的第一个元素
    • RPOP key  #移出并获取列表中的最后一个元素
    • BLPOP key1 [key2] timeout  #移出并获取列表中的第一个/多个元素,如果队列中没有元素则阻塞列表直到超时或者发现可弹出元素
    • BRPOP key1 [key2] timeout  
    • LTRIM key start stop  #删除指定区间的元素
    • LSET key index value  #通过索引设置元素值
    • LINSERT key before|after 'value' value1  #从表头开始计算，在元素'value'前或后面插入元素value1
    • RPOPLPUSH source destination  #移除列表中的最后一个元素，并元素添加到另外一个列表并返回
## Set
  String类型无序集合，集合成员唯一，不能出现重复, 类似于hashtable  
    ○ 底层使用了intset和hashtable两种数据结构，intset其实是数组，存储数据有序，采用二分法查找  
    ○ hashtable就是普通的hash表
### 常用命令
    • SCARD key  #获取集合的成员数
    • SADD key member1 [member2]  #向集合添加一个或多个成员
    • SMEMBERS key  #返回集合中的所有成员
    • SISMEMBER key member  #判断member是否是集合中的成员
    • SRANDMEMBER key [count]  # 返回集合中一个或者多个随机数
    • SREM key member [member2]  #删除集合中的一个或多个成员
    • SPOP key [count]  #移除并返回集合中的一个随机成员
    • SMOVE source destination member  #将member从source移动到destination
    • SDIFF key1 [key2]  #返回给定所有集合的差集(左侧)
    • SDIFFSTORE destination key1 [key2]  #返回给定集合的叉积并存储在destination中
    • SINTER key1 [key2]  #返回指定集合的交集
    • SINTERSTORE destination key1 [key2] 
    • SUNION key1 [key2]  #返回所有给定集合的并集
    • SUNIONSTORE destination key1 [key2]  
## Zset
  (sorted set)，redis有序集合，也是string类型的集合，且不允许重复  
    ○ 每个元素都会关联一个double类型的分数，redis通过分数来为集合成员进行从小到大的排序  
    ○ 有序集合的成员是唯一的，但分数可以重复  
    ○ 集合是通过hash table来实现的，添加、删除、查找的复杂度都是O(1)  
### 常用命令:
    • ZADD key score1 member1 [score2 member2]  #向集合中添加一个或者多个成员，或者更新已经存在的成员分数
    • ZCARD key  #或许有序集合的成员数
    • ZCOUNT key min max  #计算在有序集合中指定区间的成员数
    • ZRANK key member  #返回有序集合中指定成员的索引 
    • ZRANGE key start stop [withscores]  #返回有序集中指定区间内的成员(升序)
    • ZREVRANGE key start stop [withscores]  #返回有序集中指定区间内的成员(降序)
    • ZREM key member [member]  #删除有序集中的一个或多个成员
    • ZREMRANGEBYRANK key start stop  #移除有序集合中指定排名区间的所有元素
    • ZREMRANGEBYSCORE key start stop  #移除有序集合中指定分数区间的所有元素
# 事务
   redis事务可以一次性执行多条命令(并允许在单独的步骤中执行一组命令)，并且  
    ○ Redis会将一个事务中的命令序列化，然后按照顺序执行  
    ○ 执行过程中不会被其他命令插入，不允许出现加塞行为  
   批量操作在发送exec命令前被放入队列缓存， 收到exec命令后进入事务执行，事务中的任何命令执行失败，其余命令继续执行
<img style="margin-left:20px" src="/img/2019-08-04-redis-primary-knowledge/txn.png" />
## 常用命令
    ○ Multi 标记事务开始
    ○ Exec 执行所有事务块中的命令
    ○ Discard 取消事务，放弃执行事务块内的所有命令
    ○ Watch key1 [key2], 监视一个或者多个key，如果在事务执行之前这些key被其他命令修改，则事务被取消
    ○ Unwatch 取消watch命令监视的所有key，如果执行了watch 命令之后，exec或者discard命令被执行，那么就不需要执行unwatch了
<img style="margin-left:20px" src="/img/2019-08-04-redis-primary-knowledge/txn-sample.png" />
## 错误处理
    ○ 如果执行的某个命令报出了错误，只有报错的命令不会被执行，其他命令正常执行
    ○ 如果某条命令出现了报告错误(语法错误)，执行的整个队列会被取消
# 数据备份 - RDB
redis默认的持久化机制，RDB相当于内存快照，保存的是一种状态,将内存的数据以快照的方式写入到二进制文件中(一种压缩的二进制文件)，默认的文件名为dump.rdb  
    • save命令  
    save命令会阻塞redis服务器的进程，直到RDB文件创建完，在该期间，redis不能处理任何的命令请求，这就是save命令最大的缺陷。  
    • bgsave命令  
    与save命令不同的是，bgsave在生成RDB文件时，会派生出一个子进程，子进程负责创建RDB文件，在此期间，主进程和子进程是同时存在的，因此不会阻塞redis服务器进程  
## RDB的配置
```
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb
dir ./      #配置 RDB 文件存放的路径
stop-writes-on-bgsave-error yes    #当生成 RDB 文件出错时是否继续处理 Redis 写命令，默认为不处理
rdbcompression yes  #是否对 RDB 文件进行压缩
rdbchecksum yes  #是否对 RDB 文件进行校验和校验
```
**Pros**:  
    1）通过合理的配置，可以让Redis每隔一段时间就保存一次数据库副本，也可以很方便地将数据还原到特定的时间点。  
    2）RDB文件相比AOF占用的空间更小，恢复数据的速度也更快。  
    3）如果创建RDB文件时出现了错误，Redis不会将它用于替换原来的文件，所以出错时不会影响到之前保存的版本。  
**Cons**:  
    1）如果硬件、系统、Redis三者其中之一出现问题而崩溃，Redis会丢失全部数据，保留下来的数据只有上一个时间点创建的快照。如果数据对于应用程序来说非常重要，那么出现错误时的损失会非常大。  
    2）fork子进程占用的内存随着数据库中数据的增加而增加，耗费的时间也会越来越多  
# 数据备份 - AOF(append only file)  
  快照方式是一定的间隔时间做一次，所以redis意外down就会丢失最后一次快照的所有修改，比如900s,AOF是redis对将所有的写命令保存到一个aof文件中，根据这些写命令，实现数据的持久化和数据恢复。
## AOF配置
```
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec 
#同步频率  
  - always：redis执行每个写命令时，都同步写入硬盘，这样会严重降低redis性能  
  - everysec：每秒执行一次，显示的在这一秒内执行的写命令同步到硬盘；  
  - no: 不同步到硬盘（让操作系统来决定何时进行同步）  
```
## AOF文件生成机制
  生成过程包括三个步骤：命令追加、文件写入、文件同步。  
  redis打开AOF持久化功能之后，redis在执行完一个写命令后，把执行的命令首先追加到redis内部的aof_buf缓冲区末尾，此时缓冲区的记录还没有写到appendonly.aof文件中。
  然后，缓冲区的写命令会被写入到 AOF 文件，这一过程是文件写入过程。对于操作系统来说，调用write函数并不会立刻将数据写入到硬盘，为了将数据真正写入硬盘，还需要调用fsync函数， 调用fsync函数即是文件同步的过程，只有经过了文件的同步过程，写命令才真正的被保存到了AOF文件中。appendfsync 就是配置同步的频率的选项.
  
**Pros**:  
    1）.提供了多种同步命令的方式，默认1秒同步一次写命令，做多会丢失1秒内的数据；  
    2）.如果AOF文件有错误，比如在写AOF文件时redis崩溃了，redis提供了多种恢复AOF文件的方式，例如使用redis-check-aof工具修正AOF文件（一般都是最后一条写命令有问题，可以手动取出最后一条写命令）；  
    3）.AOF文件可读性交强，也可手动操作写命令。  
**Cons**:  
    1）.AOF文件比RDB文件较大；  
    2）.redis负载较高时，RDB文件比AOF文件具有更好的性能；  
    3）.RDB使用快照的方式持久化整个redis数据，而aof只是追加写命令，因此从理论上来说，RDB比AOF方式更加健壮，另外，官方文档也指出，在某些情况下，AOF的确也存在一些bug，比如使用阻塞命令时，这些bug的场景RDB是不存在的  
# Redis缓存与数据库一致性
## 实时同步
    ○ 查询缓存，查询不到再从DB查询保存到缓存；
    ○ 更新缓存时，先更新数据库，再将缓存设置过期
## 缓存问题
### 缓存穿透  
  是指查询一个一定不存在的key，由于缓存不明中时需要从db查询，查不到则不写入缓存，这将导致这个不存在数据每次请求都要到db去查询，造成缓存穿透.  
  解决办法：  
  • 持久层查询不到就缓存空结果，查询时先判断key是否exists，如果有就直接返回空，没有则查询后返回  
  • insert时需要清除缓存，否则db有值也不会被返回(可以设置空缓存的过期时间)  
### 缓存雪崩
  如果缓存集中在一段时间内失效，发生大量的缓存穿透，所有的查询都落在数据库上，就造成了缓存雪崩.  
  • 没有完美的解决办法，但可以分析用户行为，尽量让失效时间点均匀分布。考虑加锁或者使用队列的方式保证缓存单线写，从而避免失效时大量请求落到底层存储系统上。  
  • 缓存预热  
### 热点key  
  某些key访问非常频繁，当key失效时会有大量请求进入DB或者下游系统，导致负载增加，系统崩溃
  解决办法：  
  • 使用锁，单机用synchronized，lock等，分布式锁  
  • 缓存过期时间不设置，而是设置在value中，如果监测到超过过期时间则异步更新  
  • 在value中设置一个比过期时间t0小的过期时间t1,当t1过期时，延长t1并更新缓存  

参考文章  
https://www.jianshu.com/p/cbe1238f592a  
https://www.cnblogs.com/zhenghongxin/p/8669913.html  
https://www.bilibili.com/video/av49517046  
