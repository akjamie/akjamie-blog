---
title:       "Spring Transaction"
subtitle:    "Local Transaction"
description: "Spring Transaction 之 Local Transaction实现原理"
date:        2019-08-30
author:      "Jamie Zhang"
image:       "/img/background-03.jpg"
tags:        ["Spring", "Transaction Management"]
categories:  ["Microservice" ]
---
# 事务特性和隔离级别
## 事务基础
### 事务特性
    - 原子性(Atomicity)
    事务包含的所有操作是一个原子单元，要么全部成功，要么全部失败
    - 一致性（Consistency）
    A给B转钱，A减和B增这两个操作必须保持一致
    - 隔离性（Isolation）
    事务中的数据可见性，不同隔离级别，确保了不同的可见性级别，防止脏读，幻读等
    - 持久性（Durability）
    事务操作结果将被持久化到存储磁盘上
### 事务的隔离级别
    - Read Uncommitted，可以读取其它事务未完成的结果
    - Read Committed，在该事务完成后，才能读取该事务的数据更新后的结果
    - Repeatable Read，可以保证在整个事务的过程中，对同一笔数据的读取结果是相同的，不管其他事务是否同时在对同一笔数据进行更新，也不管其他事务对同一笔数 据的更新提交与否
    - Serializable，最严格的事务隔离控制，类似于表级别锁，所有事务操作依次有序执行
## SQL事务测试举例 - mysql
    查看事务隔离级别, 默认是REPEATABLE-READ
    select @@global.transaction_isolation,@@transaction_isolation; 
    
    设置事务隔离级别
    SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
    level: { REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED | SERIALIZABLE }
通过设置mysql的session或者全局事务隔离级别，来查看事务执行结果


以上为基本的事务知识回顾，下面我们进入spring事务。
# Spring 事务
Spring并不会直接管理事务，而是提供了事务管理器，将事务管理的职责委托给JPA JDBC JTA DataSourceTransaction JMSTransactionManager 等框架提供的事务来实现。

Spring支持两种事务，及全局事务和本地事务。
## Local transaction 及原理
本地事务，指定resource的事务，跟应用服务器无关，如DataSourceTransactionManager

spring事务的顶层接口为：PlatformTransactionManager
```
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(@Nullable TransactionDefinition var1) throws TransactionException;

    void commit(TransactionStatus var1) throws TransactionException;

    void rollback(TransactionStatus var1) throws TransactionException;
}
```
由定义可见，该接口主要提供了如上3个主要事务接口，那么他们在使用过程如何工作的呢

以springboot 2.1.x版本为例：我们一般在方法上添加@transactional去开启事务，那spring是如何解析并处理事务的呢？
### 本地事务解析配置加载过程
 - TransactionManager的加载：
  springboot工厂加载机制： 在spring-boot-autoconfigure jar包下面的spring 工厂配置文件中可以找到  
 ```
 org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration
 ```
 在这个配置类总可以找到transactionmanager的bean定义
 ```
@Bean
@ConditionalOnMissingBean({PlatformTransactionManager.class})
public DataSourceTransactionManager transactionManager(DataSourceProperties properties) {
    DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(this.dataSource);
    if (this.transactionManagerCustomizers != null) {
        this.transactionManagerCustomizers.customize(transactionManager);
    }

    return transactionManager;
}
 ```
 - 配置切面来拦截@transactional
 ```
@Configuration
public class AspectJTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

    @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ASPECT_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public AnnotationTransactionAspect transactionAspect() {
        AnnotationTransactionAspect txAspect = AnnotationTransactionAspect.aspectOf();
        if (this.txManager != null) {
            txAspect.setTransactionManager(this.txManager);
        }
        return txAspect;
    }

}
 ```  
在AnnotationTransactionAspect中，它定义了如下pointcuts
 ![](/img/2019-08-30-spring-local-transaction/txn-pointcut.png)

### 事务工作流程
在AnnotationTransactionAspect的父类AbstractTransactionAspect中定义advise - around，实现如下：
```
Object around(final Object txObject): transactionalMethodExecution(txObject) {
        MethodSignature methodSignature = (MethodSignature) thisJoinPoint.getSignature();
        // Adapt to TransactionAspectSupport's invokeWithinTransaction...
        try {
            return invokeWithinTransaction(methodSignature.getMethod(), txObject.getClass(), new InvocationCallback() {
                public Object proceedWithInvocation() throws Throwable {
                    return proceed(txObject);
                }
            });
        }
        catch (RuntimeException | Error ex) {
            throw ex;
        }
        catch (Throwable thr) {
            Rethrower.rethrow(thr);
            throw new IllegalStateException("Should never get here", thr);
        }
    }
```
由上面around advise可以看到其实主要实现在invokeWithinTransaction中，我们继续跟踪invokeWithinTransaction的实现
![](/img/2019-08-30-spring-local-transaction/txn-aspect-working-model.png)
## 手动开启事务的方式 - 非jdbc
了解了上面@transactional之后，我们再来看看如何在spring中手动开启事务
部分实例代码如下：
```
 @Autowired
private PlatformTransactionManager transactionManager;

...

@Transactional
public Customer add(Customer customer) {
    DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
    definition.setIsolationLevel(TransactionDefinition.ISOLATION_REPEATABLE_READ);
    definition.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    TransactionStatus transactionStatus = transactionManager.getTransaction(definition);

    try {
        customer = repository.save(customer);
        simulateException();
        transactionManager.commit(transactionStatus);
    } catch (Exception e) {
        transactionManager.rollback(transactionStatus);
        throw e;
    }
```
 - 注入Spring事务接口PlatformTransactionManager
 - 创建transactionDefinition,主要用来配置事务隔离级别，传播特性等
 - 调用 事务接口获取transaction
 - 根据业务执行情况进行commit/rollback方法调用

## 注意事项
在查看DataSourceTransactionManager的过程中，你会发现DataSourceTransactionObject是被绑定到ThreadLocal中的，也就是说在webflux中则无法使用。  

参考文章：
https://zhuanlan.zhihu.com/p/57746720  
https://www.jianshu.com/p/d7bfd6c1115f  





