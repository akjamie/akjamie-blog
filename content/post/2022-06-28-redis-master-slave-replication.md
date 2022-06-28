---
title: "Redis学习笔记"
date: 2022-06-28
subtitle: "Redis basic knowledge - Master/slave replication"
description: "Redis主从复制"
author: "Jamie Zhang"
image: "/img/background-05.jpg"
published: false
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
本次尝试用kubernetes来搭建redis集群 一主多从,但是不会从0开始搭建。
已有环境配置如下：

| Resource | Configs | Remark |
| :---  | :---- |  :--- |
| Master node| CPU: 2core, MEM: 2Gb|VmWare Fusion instance.|
| Worker node |CPU: 2core, MEM: 3Gb|VmWare Fusion instance.|
| Metallb | DeamonSet| Opensource load balancer.|
| Ingress-nginx| Service |Exposes HTTP and HTTPS routes from outside the cluster to services within the cluster|
| NFS| backed by worker node| Expose /opt/data as nfs root path.|
| Dashboard| Service| Dashboard with cluster admin role user access. |
| Network| Calico | Calico support network policy.|

本次会复用已有的nfs storage来创建pv和pvc，部署一主一丛 redis集群，支持自动根据内存用量扩容。
### 
本次会部署redis到dev命名空间下面





