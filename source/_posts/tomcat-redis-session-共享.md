---
title: tomcat redis session 共享
date: 2020-05-16 12:44:33
tags: "tomcat"
categories: "tomcat"
---
### 环境准备
- apache-tomcat-8.5.54
- nginx-1.17.3
- redis:5.0
### 下载jar包
- redisson-all-3.12.4.jar
- redisson-tomcat-8-3.12.4.jar

根据tomcat版本下载对应的jar包

### 添加redis配置文件

在tomcat/conf目录下，创建一个redission-single.json
```
{
   "singleServerConfig":{
      "idleConnectionTimeout":10000,
      "connectTimeout":10000,
      "timeout":3000,
      "retryAttempts":3,
      "retryInterval":1500,
      "password":"passwd",
      "subscriptionsPerConnection":5,
      "clientName":null,
      "address": "redis://192.168.1.13:6379",
      "subscriptionConnectionMinimumIdleSize":1,
      "subscriptionConnectionPoolSize":50,
      "connectionMinimumIdleSize":32,
      "connectionPoolSize":64,
      "database":0,
      "dnsMonitoringInterval":5000
   },
   "threads":0,
   "nettyThreads":0,
   "codec":{
      "class":"org.redisson.codec.JsonJacksonCodec"
   },
   "transportMode":"NIO"
}

```
### 修改conf/context.xml配置
添加
```
<Manager className="org.redisson.tomcat.RedissonSessionManager" configPath="${catalina.base}/conf/redission-single.json" readMode="REDIS" updateMode="DEFAULT"/>
```
### 参考
- [tomcat session manager](https://github.com/redisson/redisson/wiki/14.-%E7%AC%AC%E4%B8%89%E6%96%B9%E6%A1%86%E6%9E%B6%E6%95%B4%E5%90%88#145-tomcat%E4%BC%9A%E8%AF%9D%E7%AE%A1%E7%90%86%E5%99%A8tomcat-session-manager)
- [单redis节点模式](https://github.com/redisson/redisson/wiki/2.-%E9%85%8D%E7%BD%AE%E6%96%B9%E6%B3%95#26-%E5%8D%95redis%E8%8A%82%E7%82%B9%E6%A8%A1%E5%BC%8F)
