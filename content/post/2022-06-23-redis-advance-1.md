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

## 业务用例
### 整型数据排序
(这个其实不算做bitmap的一个用例场景)
思路分析：  
- bitmap是用每一个位来记录事物的状态，最大长度512Mb = 512 * 1024 * 1024 * 8 = 2^32即
8bit = 1b， offset maximum = 2^32 -1 = 4294967295,即最大的下标为4294967295
- 将整型数据直接offset设置到bitmap存储结构中，即setbit key value 1

简易示例，发现放进数据容易，取出来还是比较麻烦，不过bitmap主要用来应对状态统计之类的场景，所以设计上本也不在此。
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
在互联网场景下使用bitmap能取得出其不意的效果，
- 活跃用户统计(id为int类型)，如果活跃用户数据比较少，bitmap结构反而更加浪费存储空间
- 点赞以及统计

# 基于redis的秒杀
需要考虑的设计点：
 - 高并发下的缓存雪崩和缓存穿透
 - 超卖
 - 恶意请求
 - 链接暴露
 - 降级、限流、熔断

# Reference
https://zhuanlan.zhihu.com/p/480386998
