---
layout: post
title: "Spring Data & Data Persistent"
date: "2022-07-08"
subtitle: "Notes taken during studying spring data series - overview"
excerpt: ""
description: ""
author: "Jamie Zhang"
image: "/img/background-01.jpg"
tags:
     - Spring Data
     - JPA
categories: [ Microservice ]
---
# What is JPA?
 
"The Java Persistence API is the Java API for the management of persistence and object/relational mapping in Java EE and Java SE environments. It provides an object/relational mapping facility for the Java application developer using a Java domain model to manage a relational database."   
It's quoted from JSR(Java Specification Request) managed by Java Community Process.

This is just the definition of what's JPA, let's recall how we persist data before adopting JPA.

- Loading jdbc driver
```
Class.forName("com.mysql.jdbc.driver")
```
- Get Connection
```
Connection conn = DriverManager.getConnection(url,username, password)
```
- Create Statement or PreparedStatement
```
PreparedStatement stmt = conn.prepareStatement(sql);
# Set args
stmt.setXx
```
- Execute Query or Update
```
ResultSet rs = stmt.executeQuery()
# Iterator the rs to manual map to Java Domain object

stmt.executeUpdate() # for insert, delete or update
```

Seen from above steps, the difficulty parts are the mapping between Java Domain Object and args for Statement for update operation, and the mapping between ResultSet and Java Domain Object, that's what JPA can help, the so called ORM layer.

In Java, ORM layer is responsible for managing the conversion of Java objects/classes to interact with tables and columns in a relational database, so that the Java objects/classes can be stored and managed in a relational database.


# JPA and Hibernate
JPA's object-relational mapping was originally based on Hibernate, While JPA is not a tool or framework, it defines a set of concepts that guide implementers. And JPA has spawned many compatile tools and frameworks, Hibernate is just one of many JPA tools.

# DB manipulation tool
| Framework| Object for manipulate data| Object Factory/Manager|
|---- |--- |--- |
| Hibernate| Session | SessionFactory|
| Spring-data-JPA| EntityManager| EntityManagerFactory(LocalContainerEntityManagerFactoryBean)|
| Mybatis| SqlSession(DefaultSqlSession)|SqlSessionFactory|
