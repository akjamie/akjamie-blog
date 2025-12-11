---
layout: post
title: "微服务中的分布式事务"
date: 2019-07-23 19:40:10
subtitle: "(1st)One Distrubuted transaction use case"
excerpt: ""
description: "分享一个遇到的分布式事务case, 以及部分引申思考"
author: "Jamie Zhang"
image: "/img/background-05.jpg"
tags: 
     - Design
     - Distributed Transaction
     - Transaction Management
categories: [ Microservice ]
---
# 用户场景  
在某企业app上做Payment业务，在app端准备好request data， 如debit account number, credit account number, amount, notes, 点击submit。  
业务流程如下:  
1) 用户提交payment 请求  
2) server校验用户所在公司的每日转账限额(account level)  
3) 如果限额可以满足，则发起转账，否则reject请求  
![](/img/2019-07-24-one-distributed-transaction-user-case/overall-business-flow.png)

# 问题和挑战  
1. 如何规避重复提交
2. 如何避免分布式事务

# 解决思路  
1. 如何规避重复提交  
采用前端/后端生成UUID(it will be used as payment id), 需要缓存起来便于request进来后的校验， 当然也可以不缓存，只是校验这个ID是否已经存在，如果存在就reject请求。  
2. 如何避免分布式事务  
本例中，主要的分布式事务check point有3个，  
1)计算limit checking，在此过程中可能有新payment请求过来可能导致limit用尽或超支，这是业务上不能允许的;  
  采用乐观锁,把锁控制在payment db里面而不是对entitlement db中的transaction limit进行CRUD，这样会使整个事务处理变得复杂; 且兵并发性能有大幅度提升  
2)payment data 写入transaction history db  
  Oracle RAC事务，用spring transaction 注解，@Transactional  
3)update B/E system response - payment status(succ/fail)  
  Oracle RAC事务，用spring transaction 注解，@Transactional  
# 解决方案  
一个方案proposal如下,主要针对limit checking部分加锁， payment data写入和status update 单独分开事务控制(主要由于如果B/E执行成功了，而update status failed导致payment数据回滚会导致数据丢失)  
处理流程diagram如下:  
![](/img/2019-07-24-one-distributed-transaction-user-case/solution.jpg)

# 事后思考  
1. 如果使用非oracle 集群，可能需要引入其他中间件来确保事务一致性，该部分思考中  
2. 上例中暂时缺少重试机制，如果payment status update failed， 是否可以再次幂等尝试，因为payment id/uuid是唯一的  

参考文章:  
https://yq.aliyun.com/articles/284728
