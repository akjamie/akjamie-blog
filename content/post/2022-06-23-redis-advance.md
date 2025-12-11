---
layout: post
title: "Redis advance I"
date: "2022-06-23"
subtitle: "Redis advanced knowledge"
description: "Redis进阶知识总结"
author: "Jamie Zhang"
image: "/img/background-05.jpg"
tags: 
    - Redis
    - NoSQL
categories: [ NoSQL ]
---
# bitmap
在前面的基础回顾中提到了bitmap这种数据结构，主要用来应对string的位操作，简单高效  
适用场景：单状态统计，如每天系统用户登录情况统计，github commit.  
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

## 业务用例
### 整型数据排序/去重
(这个其实不算做bitmap的一个用例场景)  
思路分析:

-  bitmap是用每一个位来记录事物的状态，最大长度512Mb = 512 * 1024 * 1024 * 8 = 2^32, 即
8bit = 1byte， offset maximum = 2^32 -1 = 4294967295,即最大的下标为4294967295  
-  将整型数据直接offset设置到bitmap存储结构中，即setbit key value 1  

简易示例:  
发现放进数据容易，取出来还是比较麻烦，不过bitmap主要用来应对状态统计之类的场景，所以设计上本也不在此。
```
@Test  
public void test() {  
    String key = "test:bitmaps:comparison";  
    List<Integer> list = List.of(10, 39, 44, 20, 75, 63, 25, 44, 29, 21, 65, 20);  
    log.info("List before sorting,[{}]", list);  
    RedisCommands<String, String> syncCommands = connection.sync();  
  
    list.forEach(v -> {  
        syncCommands.setbit(key, v, 1);  
    });  
  
    List<Integer> sortedList = new ArrayList<>(list.size());  
    IntStream.range(1, list.stream().max(Integer::compareTo).get() + 1).forEach(i -> {  
        if (list.contains(i) && syncCommands.getbit(key, Integer.valueOf(i)).intValue() == 1) {  
            sortedList.add(i);  
        }  
    });  
    log.info("List after sorting,[{}]", sortedList);  
}
```

###  其他互联网场景
在互联网场景下使用bitmap能取得出其不意的效果，  典型用例场景如

- 活跃用户统计(id为int类型)，如果活跃用户数据比较少，bitmap结构反而更加浪费存储空间  
- 点赞以及统计  

# 基于redis的秒杀
需要考虑的设计点：

- 高并发下的缓存雪崩和缓存穿透  
- 超卖  
- 恶意请求  
- 链接暴露  
- 降级、限流、熔断  

应对方案思路：  

- 单一职责，设计单独的秒杀服务和数据库   
- 前端限流 + 后段限流 + 按钮控制  
	- 没有开始之前按钮置灰 + 延迟几秒开始 + 每次点击完之后的短暂等待=> 解决开始之前的大流量刷新以及其导致的缓存穿透  
	- 限流组件(hystrix/sentinel) + 及时秒杀结束(前后端)  
- 防刷控制  
	- 动态URL，商品链接加盐或者加密  
	- 恶意ip拦截，请求次数>阈值  
	- 秒杀成功白名单，保证每个id最多秒杀成功一次  
- 流量洪峰应对，保证系统运行  
	- nginx+预热扩容+结束后的缩容  
	- 资源静态化 - cdn缓存  
- 存储预热  
	- redis集群  
	- 商品库存和秒杀开关缓存到redis
	- redis事务解决库存判断,扣减和秒杀开关更新 ->lua 脚本
- 超卖控制
	- redis lua 脚本 - redis事务  
	- 数据库秒杀记录唯一索引 （redis扣减成功但数据库更新失败的case可以忽略不计，对商家来说无损失，反正已经宣传了自己）  
- 秒杀成功的订单入队列异步处理后续流程，如邮寄地址/付款/etc.  

动态url示例: 

1. 获取url  --> GET:/{goodId}/url, 返回的是混淆字符串(hash)，如sha2(goodId+salt)
2. 秒杀下单--> POST:/{goodId}/{hash}/execution. 比较hash是否正确。

# 数据缓存
记录一个分布式缓存的用例场景  
需求： 一个债券订单系统， 为了提升债券的检索性能(毫秒级返回)以提升用户体验，债券信息检索方案由直查db替换为缓存债券信息到redis缓存。  

主要设计点：  
1. 每天后台定时任务多线程更新缓存，如工作日6:00am  
2. 分布式锁，确保只有一个实例负责更新缓存  
3. <String,String> key,value 存储，key中包含常用搜索字段，如债券搜索的常用字段为债券id，货币类型，债券名称，那么对应的redis key设计为，bond:{bondId}:{bondName}:{currency}, value则为债券对象的json    
4. 用keys 或者scan来进行模糊查询  
	- keys命令是阻塞的方式  
	- 推荐使用scan
	
```
/*
**Input**
127.0.0.1:6379> keys test-string-*
1) "test-string-1"
2) "test-string-2"
3) "test-string-3"
127.0.0.1:6379>
*/
	
// ---------------------------------------
@SneakyThrows  
@Test  
public void test() {  
    String queryKey = "test-string-*";  
    RedisCommands<String, String> syncCommands = connection.sync();  
    List<String> keys = new ArrayList<>();  
    List<String> values = new ArrayList<>();  
    ScanArgs scanArgs = new ScanArgs();  
    scanArgs.match(queryKey);  
    scanArgs.limit(2);  
    KeyScanCursor<String> scanCursor = null;  
    do {  
        if (scanCursor == null) {  
            scanCursor = syncCommands.scan(scanArgs);  
        } else {  
            scanCursor = syncCommands.scan(scanCursor, scanArgs);  
        }  
  
        // get all page per page  
        keys.addAll(scanCursor.getKeys());  
    }while(!scanCursor.isFinished());  
  
    //todo: to replace it with pipelines  
    keys.forEach(k -> {  
        values.add(syncCommands.get(k));  
    });  
  
    log.info("Matched records, count: {}", values.size());  
    values.forEach(System.out::println);  
}
	
// -------------------------------
**Output**
23:52:44.141 [main] INFO org.akj.redis.ScanCommandTest - Matched records, count:3
this is for quick search testing
this is testing message
redis
```

# 典型缓存问题
## 缓存穿透
缓存中无数据，数据库中也无数据  
如恶意攻击，使用没有的key访问导致数据库压力过大。  
应对办法：  
- 通过鉴权，过滤不合理的请求  
- 查询不到的数据加入缓存，value为空切设置过期时间 （不常用，无法防止动态key导致的缓存浪费）  
- bitmap或者bloomer filter.

## 缓存击穿
缓存中无数据，但是数据库中有数据  
高并发下hot key过期导致大量请求流到数据库，导致数据库访问过载  
应对办法：
- ttl hot-key 不过期  
- 后台线程更新缓存数据且分布式加锁互斥

## 缓存雪崩
缓存中无数据，但是数据库中有数据  
高并发下，缓存同一时刻失效（down机或者设置了相同的过期时间），所有请求都会到数据库导致数据负载过高或者down机，重启后可能立即又down机。  
应对办法：
- 缓存失效时间设置为随机值，避免同时失效  
- redis clsuter， 或者主从+sentinel  
- 服务限流，降级，避免数据库被压垮  


# Reference
https://zhuanlan.zhihu.com/p/480386998
