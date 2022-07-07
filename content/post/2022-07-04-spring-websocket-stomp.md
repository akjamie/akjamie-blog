---
title: "Websocket集成"
date: 2022-07-04
subtitle: "Mark down spring stomp & related websocket knowledge"
description: "记录websocket集成的杂事"
author: "Jamie Zhang"
image: "/img/background-05.jpg"
published: true
tags: ["Websocket", "Spring-stomp", "K8S"]
categories: [ Microservice ]
---

# Background
最近在做一个类似于快速竞价的功能的技术设计，即在很短的时间内通过一个集中的平台拿到各家的报价，类似于支付宝上的那个车险报价，发送完汽车信息，得到各家保险公司的报价回复，然后可以选择某家性价比高的价格购买汽车保险。在设计的过程很快就想到了用websocket+stomp这种长链接和event-streaming 推送的方式，做了一些技术研究和验证，因此便记录这些零碎的知识便于以后查阅。

# Reffering articles
1.偶然找到了下面这篇文章，写的挺详细的便引用过来，弥补了spring官方文档不够详细的不足。  
https://medium.com/swlh/websockets-with-spring-part-1-http-and-websocket-36c69df1c2ee  
https://medium.com/swlh/websockets-with-spring-part-2-websocket-with-sockjs-fallback-1903cf8fe480  
https://medium.com/swlh/websockets-with-spring-part-3-stomp-over-websocket-3dab4a21f397
