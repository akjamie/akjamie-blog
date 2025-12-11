---
layout: post
title: Linux network proxy setup for desktop and terminal
date: 2019-07-14 14:50:52
subtitle: "setup network proxy on aws(free tier is enough)"
excerpt: ""
description: "Using Shadowsocks+proxychain to build network proxy for personal learning purpose"
author: "Jamie Zhang"
image: "/img/background-01.jpg"
tags:
     - Shadowsocks
     - Proxychain
URL: "/post/2019-07-14-linux-network-proxy-setup-desktop-terminal/"
categories: [ Others ]
---
# 背景介绍
很多时候我们由于great firewall，我们无法访问某些网站，这时候就需要用到网络代理来应对一些比较urgent的case，本文主要会介绍如何使用Shadowsocks 和 Proxychain来 setup linux network proxy，这样在命令行下都可以起到代理和转发的作用，而不仅限于浏览器的代理。

# SS server & SS local的配置
## 预置条件
### Installed Python 3 + pip
安装命令如下:  
```
sudo apt-get install python3-pip
```
*如果得到no packages found - python3-pip的错误，请添加相应的apt sources. 修改/etc/apt/sources.list文件，添加如下4条source信息*
```
  deb http://cn.archive.ubuntu.com/ubuntu bionic main multiverse restricted universe
  deb http://cn.archive.ubuntu.com/ubuntu bionic-updates main multiverse restricted universe
  deb http://cn.archive.ubuntu.com/ubuntu bionic-security main multiverse restricted universe
  deb http://cn.archive.ubuntu.com/ubuntu bionic-proposed main multiverse restricted universe
```

### Installed Shadowsocks
### 一台可以访问__外网__的云主机，Azure/AWS/Alicloud-HK等都可以
```
sudo pip install shadowsocks
```
## SSServer的配置和启动
在云主机上创建sss 配置文件shadowsocks.json(推荐放置在/etc/shadowsocks/ 下面)，其内容如下：
```
{
  "server":"0.0.0.0",
  "server_port":"{port}",
  "local_address":"127.0.0.1",
  "local_port":"1080",
  "password":"{password}",
  "timeout":1000,
  "method":"aes-256-cfb",
  "fast_open":false
}
```
ip,port,password这三个参数是必须参数
配置好这个配置文件后，运行命令然SSS服务运行起来
```
nohup ssserver -c /etc/shadowsocks/shadowsocks.json start &
```
## SSLocal的配置和启动
在配置好了服务端之后，我们就可以开始配置客户端了，客户端也需要一个基本类似的配置文件，当然了你可以在windows或者手机上用shadowsocks的客户端来实现客户端代理，这个不是本文的重点
在本地的linux机上创建shadowsocks.json，配置内容如下：
```
{
  "server":"xxxx",
  "local_address": "127.0.0.1",
  "local_port":1080,
  "server_port":xxxx,
  "password":"xxxx",
  "timeout":300,
  "method":"aes-256-cfb"
}
```
完成配置后，命令运行起来
```
nohup sslocal -c shadowsocks.json &
```

到此，我们可以通过设置linux系统环境变量达到浏览器network代理的目的,这距离我们的最终目标命令行下的网络代理更近一步了
>export http_proxy=127.0.0.1:1080
export https_proxy=127.0.0.1:1080
export no_proxy=localhost, 127.0.0.1

接下来我们进行proxychain的安装和配置
# Proxychain的配置
## Install proxychain
>
git clone https://github.com/rofl0r/proxychains-ng  
cd proxychains-ng  
./configure --prefix=/usr --sysconfdir=/etc  
make  
make install  
make install-config  

按步骤执行如上命令完成后，proxychain安装完成，并且默认的配置为：/etc/proxychains.conf

## Update proxychain的代理列表
> vim /etc/proxychains.conf
在文件尾部增加一行：socks5    127.0.0.1  1080
<img src="/img/2019-07-14-linux-network-proxy-setup-desktop-terminal/15532408-1405d25f0bcee21b.png" style="margin-left:25px" />

现在你可以测试了，测试命令如下：
>proxychains4 curl https://www.google.com

如果你能看到服务器返回了html相关的东西，说明代理是工作的，恭喜你
截至目前，你可以做到在命令下来代理网络了，可以应对很多的使用场景了，但是依然有场景你无法满足，例如，在某个应用的内需要走代理，前提是你无法添加proxychains4的前缀，如docker daemon里面去下载image，这时候需要一个‘全局’的代理，使到全部的流量都走代理，不管是inbound或者outbound，你会问可以做到吗？
答案是Yes，很简单，只需要一个命令去打开bash并在这个bash下面所有的流量都走代理
> proxychains4 -q /bin/bash

到此为止，我们基本实现了基于浏览器代理和命令行下的全局代理，自己去试试吧，good luck

最后一点补充，可以把SS做成linux服务，这样随系统启动使用起来会更加的方便，做法如下：
>sudo vim /etc/systemd/system/shadowsocks.service
内容如下：
*[Unit]
Description=Shadowsocks Client Service
After=network.target
[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/sslocal -c /xxx/shadowsocks.json
[Install]
WantedBy=multi-user.target*

激活SS服务随系统自动启动
>systemctl enable /etc/systemd/system/shadowsocks.service

然后就可以像其他linux服务那样使用 systemctl来管理了
如果在上述添加SS服务的过程中遇到这个错误的话
/usr/lib/x86_64-linux-gnu/libcrypto.so.1.1: undefined symbol: EVP_CIPHER
请修改/usr/local/lib/python3.6/dist-packages/shadowsocks/crypto/openssl.py，更新第55行和111行
将libcrypto.EVP_CIPHER_CTX_cleanup(self._ctx) 修改为 <label style="color:blue">libcrypto.EVP_CIPHER_CTX_reset(self._ctx)</label>
