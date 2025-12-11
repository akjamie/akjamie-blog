---
layout: post
title:       "Spring Transaction"
subtitle:    "Global Transaction"
description: "Spring Transaction 之 JTA事务管理"
date:        2019-08-31
author:      "Jamie Zhang"
image:       "/img/background-03.jpg"
tags:        ["Spring", "Transaction Management"]
categories:  ["Microservice" ]
---
先回顾上一篇sprint local transaction的内容，本地事务是用于指定资源，即单一数据源，其主要结构如下：
 ![](/img/2019-08-31-spring-global-transaction/spring-txn-local.png)
 那么我们来看看spring的global transaction管理，其主要用于多数据源的业务场景中，如mysql + mysql, oracle + mysql + rabbitmq等，但是说到全局事务，我们需要先来谈谈xa。

# XA 与 JTA
## 什么是XA
```
XA 是指由 X/Open 组织提出的分布式事务处理的规范. XA 规范主要定义了事务管理器(Transaction Manager)和局部资源管理器(Local Resource Manager)之间的接口.
目前，Oracle、Informix、DB2和Sybase等各大数据库厂家都提供对XA的支持。XA协议采用两阶段提交方式来管理分布式事务。
```
由如上的信息中我们能看到，在xa的规范中定义了如下3个方面：  
1. Transaction manager, 统筹管理多个resource manager  
2. Resource manager, 管理具体的XA resource  
3. Two-phase commit, 两阶段提交的方式  
_4. Xid, 事务ID 用于区分事务_
## XA与JTA的联系
jta是xa规范在java中的实现
<img src='/img/2019-08-31-spring-global-transaction/xa-jta.png' style="height: 317px;margin-left: 15px;"/>
# 全局/JTA事务
## JTA事务的两种支持方式
1. 基于j2ee 应用服务器的jta，事务管理器和xa资源管理器均由应用服务器来管理，如jboss, ibm web sphere
2. 采用非应用服务器的方式，如采用atomikos,Bitronix等
![](/img/2019-08-31-spring-global-transaction/spring-txn-global.png)
## Atomikos 配置多数据源支持jta事务
环境准备：  
. 两台mysql数据库
主要配置：  
. datasource 配置文件
```
spring:
  datasource:
    user:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://localhost:3306/xa?allowPublicKeyRetrieval=true&useSSL=false&charset=utf8
      username: xxx
      password: xxx
    audit:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://xxx.rds.amazonaws.com:3306/xa?allowPublicKeyRetrieval=true&useSSL=false&charset=utf8
      username: xxx
      password: xxx
```
. 核心配置bean - transaction interface
```
@Bean(name = "userTransaction")
public UserTransaction userTransaction() throws Throwable {
    UserTransactionImp userTransactionImp = new UserTransactionImp();
    userTransactionImp.setTransactionTimeout(10000);
    return userTransactionImp;
}

@Bean(name = "atomikosTransactionManager")
public TransactionManager atomikosTransactionManager() throws Throwable {
    UserTransactionManager userTransactionManager = new UserTransactionManager();
    userTransactionManager.setForceShutdown(false);
    AtomikosJtaPlatform.transactionManager = userTransactionManager;
    return userTransactionManager;
}

@Bean(name = "transactionManager")
@DependsOn({"userTransaction", "atomikosTransactionManager"})
public PlatformTransactionManager transactionManager() throws Throwable {
    UserTransaction userTransaction = userTransaction();
    AtomikosJtaPlatform.transaction = userTransaction;
    TransactionManager atomikosTransactionManager = atomikosTransactionManager();

    return new JtaTransactionManager(userTransaction, atomikosTransactionManager);
}
```
. 核心配置bean datasource - user,需要指定一组为@primary，高优先级装配的的datasource，否则spring在初始化transaction manager的时候会报错。
```
@Bean
@Primary
@ConfigurationProperties(prefix = "spring.datasource.user")
public org.springframework.boot.autoconfigure.jdbc.DataSourceProperties customerDataSourceProperties() {
    return new org.springframework.boot.autoconfigure.jdbc.DataSourceProperties();
}

@Bean(name = "customerDataSource")
@Primary
public DataSource customerDataSource() throws SQLException {
    MysqlXADataSource mysqlXaDataSource = new MysqlXADataSource();
    mysqlXaDataSource.setURL(customerDataSourceProperties().getUrl());
    mysqlXaDataSource.setUser(customerDataSourceProperties().getUsername());
    mysqlXaDataSource.setPassword(customerDataSourceProperties().getPassword());
    mysqlXaDataSource.setPinGlobalTxToPhysicalConnection(true);

    AtomikosDataSourceBean xaDataSource = new AtomikosDataSourceBean();
    xaDataSource.setXaDataSource(mysqlXaDataSource);
    xaDataSource.setUniqueResourceName("customerDataSource");
    xaDataSource.setMaxPoolSize(100);
    xaDataSource.setMinPoolSize(20);
    return xaDataSource;
}

@Bean(name = "customerEntityManager")
@DependsOn("transactionManager")
@Primary
public LocalContainerEntityManagerFactoryBean customerEntityManager() throws Throwable {
    HashMap<String, Object> properties = new HashMap<String, Object>();
    properties.put("hibernate.transaction.jta.platform", AtomikosJtaPlatform.class.getName());
    properties.put("javax.persistence.transactionType", "JTA");

    LocalContainerEntityManagerFactoryBean entityManager = new LocalContainerEntityManagerFactoryBean();
    entityManager.setJtaDataSource(customerDataSource());
    entityManager.setJpaVendorAdapter(jpaVendorAdapter);
    entityManager.setPackagesToScan("org.akj.springboot.tx.entity.customer");
    entityManager.setPersistenceUnitName("customerPersistenceUnit");
    entityManager.setJpaPropertyMap(properties);

    return entityManager;
}
```
. 核心配置bean datasource - audit
```
@Configuration
@EnableJpaRepositories(basePackages = "org.akj.springboot.tx.entity.audit", entityManagerFactoryRef =
        "auditEntityManager", transactionManagerRef = "transactionManager")
@DependsOn("transactionManager")
public class AuditDataSourceConfiguration {

    @Autowired
    private JpaVendorAdapter jpaVendorAdapter;

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.audit")
    public org.springframework.boot.autoconfigure.jdbc.DataSourceProperties auditDataSourceProperties() {
        return new org.springframework.boot.autoconfigure.jdbc.DataSourceProperties();
    }

    @Bean(name = "auditDataSource")
    public DataSource auditDataSource() throws SQLException {
        MysqlXADataSource mysqlXaDataSource = new MysqlXADataSource();
        mysqlXaDataSource.setURL(auditDataSourceProperties().getUrl());
        mysqlXaDataSource.setUser(auditDataSourceProperties().getUsername());
        mysqlXaDataSource.setPassword(auditDataSourceProperties().getPassword());
        mysqlXaDataSource.setPinGlobalTxToPhysicalConnection(true);

        AtomikosDataSourceBean xaDataSource = new AtomikosDataSourceBean();
        xaDataSource.setXaDataSource(mysqlXaDataSource);
        xaDataSource.setUniqueResourceName("auditDataSource");
        xaDataSource.setMaxPoolSize(100);
        xaDataSource.setMinPoolSize(20);
        return xaDataSource;
    }

    @Bean(name = "auditEntityManager")
    @DependsOn("transactionManager")
    public LocalContainerEntityManagerFactoryBean auditEntityManager() throws Throwable {
        HashMap<String, Object> properties = new HashMap<String, Object>();
        properties.put("hibernate.transaction.jta.platform", AtomikosJtaPlatform.class.getName());
        properties.put("javax.persistence.transactionType", "JTA");

        LocalContainerEntityManagerFactoryBean entityManager = new LocalContainerEntityManagerFactoryBean();
        entityManager.setJtaDataSource(auditDataSource());
        entityManager.setJpaVendorAdapter(jpaVendorAdapter);
        entityManager.setPackagesToScan("org.akj.springboot.tx.entity.audit");
        entityManager.setPersistenceUnitName("auditPersistenceUnit");
        entityManager.setJpaPropertyMap(properties);

        return entityManager;
    }
}
```
*@EnableJpaRepositories, 仓储bean的扫描和注册，并且使用了这个之后，两个db都能支持auto ddl*
## Track atomikos的工作流程
JTA的事务加载和切面拦截流程跟local transaction是一样的，本次过重分析一下atomikos的两阶段提交的工作流程和重点代码分析：  
1. 搞清楚UserTransaction, TransactionManager 和 Transaction  
 - TransactionManager, 将应用对分布式事务的使用映射到实际的事务资源并在事务资源间进行协调与控制  
 - Transaction, 代表了一个物理意义上的事务，在开发人员调用 UserTransaction.begin() 方法时 TransactionManager 会创建一个 Transaction 事务对象（标志着事务的开始）并把此对象通过 ThreadLocale 关联到当前线程。**UserTransaction 接口中的 commit()、rollback()，getStatus() 等方法都将最终委托给 Transaction 类的对应方法执行**  
 - UserTransaction, 面向开发人员的接口, 直接使用该接口实现JTA事务管理  
2. 查看启动日志，观察atomikos下的相关事务配置  
o.s.t.jta.JtaTransactionManager          : Using JTA UserTransaction: **com.atomikos.icatch.jta.UserTransactionImp@5ba90d8a**  
2019-09-07 18:02:39.918 DEBUG 24695 --- [           main] o.s.t.jta.JtaTransactionManager          : Using JTA TransactionManager: **com.atomikos.icatch.jta.UserTransactionManager@4d93f75b**  
接着会打印AtomikosDataSoureBean的配置，即初始化我们在上面的datasource配置bean  
3. 部分源码分析 
```
org.springframework.transaction.interceptor.TransactionAspectSupport#createTransactionIfNecessary
    // 获取trsanction
    org.springframework.transaction.support.AbstractPlatformTransactionManager#getTransaction
        // 获取<JtaTransactionObject>transaction
        org.springframework.transaction.jta.JtaTransactionManager#doGetTransaction --> return UserTransactionImpl
            org.springframework.transaction.jta.JtaTransactionManager#doGetJtaTransaction  --> wrap up the UserTransactionImpl as JtaTransactionObject
        // 
        org.springframework.transaction.support.AbstractPlatformTransactionManager#isExistingTransaction --> txObject.getUserTransaction().getStatus() != Status.STATUS_NO_TRANSACTION

        // 创建事务TransactionStatus
        DefaultTransactionStatus status = new TransactionStatus(
                        definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);

        // 开启事务
        doBegin(transaction, definition);
            org.springframework.transaction.jta.JtaTransactionManager#doBegin
                // 设置隔离机制和timeout
                org.springframework.transaction.jta.JtaTransactionManager#doJtaBegin
                    // 检查transationmanager的初始化
                    com.atomikos.icatch.jta.UserTransactionImp#begin
                        com.atomikos.icatch.jta.TransactionManagerImp#begin
                            com.atomikos.icatch.imp.CompositeTransactionManagerImp#createCompositeTransaction
                            // 创建 CompositeTransaction， 并
                            ret = getTransactionService().createCompositeTransaction ( timeout )
                                - 检查maxNumberOfActiveTransactions < maxNumberOfActiveTransactions.  <-- com.atomikos.icatch.max_actives= 300 in jta.properties
                                - 获取tid
                                - CoordinatorImp cc = createCC ( null, tid, true, false, timeout ) 创建事务协调器
                                - 创建 CompositeTransaction

                                com.atomikos.icatch.imp.CompositeTransactionManagerImp#setThreadMappings
                                    // 用hashmap保存了线程和CompositeTransaction的关系
                                    if ( TxState.ACTIVE.equals ( ct.getState() )) {
                                        Stack<CompositeTransaction> txs = threadtotxmap_.get ( thread );
                                        if ( txs == null )
                                            txs = new Stack<CompositeTransaction>();
                                        txs.push ( ct );
                                        threadtotxmap_.put ( thread, txs );
                                        txtothreadmap_.put ( ct, thread );
                                    }
                            // 创建 TransactionImp，保存tid和transactionImpl的关系
                            com.atomikos.icatch.jta.TransactionManagerImp#recreateCompositeTransactionAsJtaTransaction
```

<p>
4. 过程日志分析  
 - Create transaction  
 - Participant in existing transaction for 'customer' datasource  
 - XAResource.start for 'customer'  
 - XAResource.end for 'customer'  
 - Participant in existing transaction for 'audit' datasource  
 - XAResource.start for 'audit'  
 - XAResource.end for 'audit'  
 - Initiating transaction commit  
 - XAResource.prepare for 'customer'  
 - XAResource.prepare for 'audit'  
 - XAResource.commit for 'customer'  
 - XAResource.commit for 'audit'  
<img src="/img/2019-08-31-spring-global-transaction/txn-01.png" style="margin-left: 15px"/>
<img src="/img/2019-08-31-spring-global-transaction/txn-02.png" style="margin-left: 15px"/>
</p>
 参考文章:  
 https://www.ibm.com/developerworks/cn/java/j-lo-jta/index.html  


