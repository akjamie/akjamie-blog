---
layout: post
title: "Spring Cloud Series"
date: 2019-07-23 13:36:10
subtitle: "(1st) Demo Overview"
excerpt: ""
description: "通过一个demo，来综合运用spring cloud系列技术"
author: "Jamie Zhang"
image: "/img/background-05.jpg"
tags:
     - Spring Cloud
     - Spring Boot
     - Docker
categories: [ Microservice ]
---

# Demo的介绍
本文主要通过一个小demo，来综合运用spring cloud系列技术，demo中主要包含如下模块/组建：
1. Authorization service, 基于Spring Security Oauth2 + Spring Cloud Oauth2,提供集中授权服务,如Token 生成,校验等  
2. API gateway, edge service, 提供统一的对外访问接口，集成了oauth2 安全校验，Netflix Hystrix服务降级，依赖隔离，断路保护功能  
3. Service registry,基于Netflix Eureka的注册中心提供服务注册和服务发现功能  
4. Config service, 采用了 Spring cloud config + Spring Cloud Bus提供简单的配置中心  
5. Catalog service, 示例服务，模拟一个简单的商品服务目录，仅一维结构  
6. Inventory service, 示例服务, 模拟简单的商品库存服务
7. Hystrix dashboard, 聚合hystrix的metrics监控展示

# 整体架构
![](/img/2019-07-15-spring-cloud-demo-overview/architecture.png)

# Spring boot 项目列表
1. authorization-service，https://github.com/cloud-poc/authorization-service
   backed by mysql database
2. cloud-api-gateway，https://github.com/cloud-poc/api-gateway
   目前主要集成了oauth2 jwt授权认证，无valid token直接reject
3. service-registry，https://github.com/cloud-poc/service-registry
   eureka 服务端，外置话了部分参数，便于在docker/docker-compose部署时根据需要配置
4. config-service，https://github.com/cloud-poc/config-service
   基于spring cloud config的配置中心，后续会考虑用携程apollo替换
5. catalog-service， https://github.com/cloud-poc/catalog-service
   模拟一个简单的商品服务目录，用到了feign和netflix robin等框架
6. inventory-service， https://github.com/cloud-poc/inventory-service
7. hystrix-turbine-dashboard，https://github.com/cloud-poc/hystrix-turbine-dashboard
 hystrix的集中dashboard， 目前turbine stream部分还有点问题，能从mq总拿到stream信息，但是dashboard展示有问题
8. devops-config-service-repo， https://github.com/cloud-poc/devops-config-service-repo
 服务configs目录，主要存放cloud configs和docker compose files
9. api-common, https://github.com/cloud-poc/api-common
 部分common代码，如generic exception的定义和controler advice，swagger的配置

# 其他
主要项目都提供了docker打包的plugin，直接运行mvn clean compile jib:dockerBuild就能实现local build，如需要推送到远程docker registry，需要配置远端仓库信息。 

在gateway，catalog，inventory中都配置了调用链跟踪，抽样数据会汇总到zipkin上进行展示，目前是基于http 方式推送数据，后面会修改为rabbit mq方式推送，提高性能 

在gateway，catalog，inventory中都配置了prometheus的监控埋点，metrics会被拉取到prometheus上，并在kafana中展示，在deveops代码仓库中提供了一个简单的prometheus+alertmanager+kafana的docker compose file

logging的统一收集和分析还没有开始
