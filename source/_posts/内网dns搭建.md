---
title: 内网dns搭建
date: 2020-05-16 15:21:39
tags: dns
categories: dns
---
### 环境
- centos 7
- docker-ce 19.03
- sameersbn/bind:latest
- 本机dns服务器的IP是192.168.1.20
### 启动bind服务
```
# docker run -d --name=bind --dns=114.114.114.114 --dns=8.8.8.8 -p 53:53/udp \
 -p 10000:10000  --volume=/srv/docker/bind:/data  --env='ROOT_PASSWORD=password' \
   sameersbn/bind:latest
```
### 修改bind配置
通过浏览器打开https://192.168.1.20:10000 用户名是root；
打开Servers=>BIND DNS Server=>Zone Defaults=>Default zone settings，选择Listed，然后文本域输入any；
然后重启bind容器；
```
# docker restart bind
```
### 通过bind-utils测试
安装bind-utils 工具
```
# yum install bind-utils
```
测试dns服务是否启动成功
```
# host www.jd.com 192.168.1.20
```

### 添加dns记录
创建zone；
<img src="create_zone.jpg" width="100%" height="100%">
保存zone；
<img src="save_zone.jpg" width="100%" height="100%">
zone列表；
<img src="zone_list.jpg" width="100%" height="100%">
添加子域名解析；
<img src="add_record.jpg" width="100%" height="100%">
保存子域名解析；
<img src="save_record.jpg" width="100%" height="100%">
重启bind容器
```
# docker restart bind
```
测试是否解析成功；
```
# host test.example.com 192.168.1.20
```
 
### 参考
- [Deploying a DNS Server using Docker](http://www.damagehead.com/blog/2015/04/28/deploying-a-dns-server-using-docker/)