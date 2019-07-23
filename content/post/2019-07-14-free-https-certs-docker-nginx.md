---
layout: post
title: "Free CA Certs setup using Certbot + Docker + Nginx"
subtitle: ""
excerpt: ""
description: "介绍用如何使用Let's Encrypt's Certbot生成CA证书，并配置在nginx中保护站点"
date: 2019-07-14 18:30:05
author: "Jamie Zhang"
image: "/img/background-02.jpg"
published: true
tags:
     - Docker
     - Let’s Encrypt
     - Nginx
URL: "/2019/07/14/free-https-certs-docker-nginx/"
categories: [ Tech ]
---
# Background
**Let's Encrypt** is a [certificate authority](https://en.wikipedia.org/wiki/Certificate_authority "Certificate authority") that provides [X.509](https://en.wikipedia.org/wiki/X.509 "X.509") [certificates](https://en.wikipedia.org/wiki/Public_key_certificate "Public key certificate") for [Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security "Transport Layer Security") (TLS) encryption at no charge,The certificate is valid for 90 days, during which renewal can take place at anytime.
这样我们就可以用上免费的CA cert来安全expose我们自己的网站或者服务

**基本的http和https**知识请阅读https://linuxstory.org/deploy-lets-encrypt-ssl-certificate-with-certbot/的‘背景知识’部分，作者讲述的非常很不错。
# Objectives
通过例子来demo如何生成和使用Internet Security Research Group推出的Let’s Encrypt 免费证书
主要涉及如下:  
1.  Docker, docker-compose用来部署nginx  
2.  Certbot，用来为域名生成CA证书

# Not In Scope
Docker和docker compose的相关概念和安装，请参考[docker官方文档](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

# Steps to configure certs
## Generate CA cert
### 安装Certbot 客户端
```
wget https://dl.eff.org/certbot-auto
chmod a+x ./certbot-auto
./certbot-auto --help
```
### 验证域名所有权
   该步骤需要启动nginx:  
a. 准备docker-compose file
```
version: '3.0'
services:
  nginx:
    restart: always
    image: nginx:1.15.6
    ports:
     - 80:80
     - 443:443
    volumes:
     - ./conf.d:/etc/nginx/conf.d
     - ./log:/var/log/nginx
     - ./wwwroot:/var/www
     - /etc/letsencrypt:/etc/letsencrypt
```
>  **Docker volume的映射关系**
./conf.d nginx的配置所在
./log 日志文件位置
./wwwroot 项目路径
/etc/letsencrypt CA cert的父目录

b. 修改nginx 配置文件 .conf.d/app.conf
```
server {
    listen   80;
    server_name   domain.com;
    location ^~ /.well-known/acme-challenge/ {
       default_type "text/plain";
       root     /var/www;
    }
    location = /.well-known/acme-challenge/ {
       return 404;
    }
}
```
> PS:
如上两个location配置是为了通过 Let’s Encrypt 的验证，验证域名归属

c. 启动nginx
```
root@aws-techpoc-c02:~/web# ls conf.d/app.conf
root@aws-techpoc-c02:~/web# ls wwwroot/
index.html
root@aws-techpoc-c02:~/web# docker-compose up -d
Creating network "web_default" with the default driver
Creating web_nginx_1 ...
Creating web_nginx_1 ... done
```
d. 生成Cert
```
./certbot-auto certonly -d domain1.com domain2.com
```
*该步骤过程中会自动运行和安装很多linux的以来包，不用干预，其中有两布需要注意：
i.选择用standalone的方式运行还是webroot，一般80端口已经备用了，无法使用standalone，所以选择webroot方式，然后输入webroot的地址，及上面指定的主机项目目录，如本例，/root/web/wwwroot
2.有一步要求输入邮箱地址的提示，照着输入自己的邮箱即可，顺利完成的话，屏幕上会有提示信息。
最后，证书成功成功后，会有如下信息:*  
IMPORTANT NOTES:
 Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/**<domain.com>**/fullchain.pem. Your cert
   will expire on xxxx-xx-xxxx. To obtain a new version of the
   certificate in the future, simply run Let's Encrypt again.
至此，证书就生成好了，接下来就可以去配置nginx ssl监听了

## 配置SSL监听 for Nginx
a. 修改nginx配置文件
```
server {
    listen   443 ssl;
    server_name  domain.com;
    ssl_certificate        /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key    /etc/letsencrypt/live/domain.com/privkey.pem;

    location / {
        root /var/www;
        index index.jsp index.html index.htm index.php;
    }

    location /proxy/ {
        root /var/www;
        index index.jsp index.html index.htm index.php;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;

        proxy_pass http://172.19.0.1:8080/;
    }

    error_page   500 502 503 504  /50x.html;

    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}

server {
    listen   80;
    server_name   domain.com www.domain.com;
    return 301 https://$server_name$request_uri;            
}
```
> proxy_pass 指定代理地址，注意代理地址后面有没有’/‘,区别很大

b. [optional]准备一个测试的html，用来检测nginx配置是否正常，./wwwroot/index.html
```
<!DOCTYPE html>
<html>
   <head>
     <title>TEST WebSite</title>
   </head>    
   <body>
       <div>Hello, this is a test web site</div>
    <body>
</html>
```
c. Reload nginx config
```
docker container exec <container> nginx -s reload
```
然后就可以去测试了，https://<domain.com>,如果成功可以显示那个test html page，如果有问题，请使用docker logs -f <container>,或者查看日志目录先access.log 和 error.log

# Cert Renew
Let’s Encrypt 签发的证书有效期只有 90 天，所以需要每隔三个月就要更新一次安全证书,为了能让你的网站能被安全的保护起来，所以是非常有必要去更新cert的。certbot-auto 支持手动和自动更新两种cert更新模式，我们来测试一下这两种模式
首先运行测试一下能否更新,调试模式，不是真的更新
```
./certbot-auto renew --dry-run
```
获得的结果截图如下
<img src="/img/2019-07-14-free-https-certs-docker-nginx/20190714194434.png" style="margin-left:20px" />

手动更新模式：
```
./certbot-auto renew -v
```
自动更新模式：
```
./certbot-auto renew --quiet --no-self-upgrade
```
> PS:该部分一般有运维人员集中进行处理，CA证书自动生成后还需要被加载一下去刷新，在本例中，需要运行docker container exec <container> nginx -s reload 去刷新

更新后的结果展示
<img src="/img/2019-07-14-free-https-certs-docker-nginx/20190714195417.png" style="margin-left:20px" />

Q&A:
1. policy-forbids-issuing-for-name-on-amazon-ec2-domain
[issue related post](https://community.letsencrypt.org/t/policy-forbids-issuing-for-name-on-amazon-ec2-domain/12692/3)  
amazonaws.com happens to be on the blacklist Let’s Encrypt uses for high-risk domain names，及无法使用free cert for aws，可以考虑使用route53

Related posts:
https://linuxstory.org/deploy-lets-encrypt-ssl-certificate-with-certbot/
https://www.jianshu.com/p/c136c7ec2572