---
title: jar 修改配置文件
date: 2021-01-20 14:55:49
tags:
---

示例：weather.jar


## 解压jar

```bash
jar -xvf weather.jar
```

## 添加或替换配置


目录结构如下
```bash
[root@mqtt weather]# ls
BOOT-INF  META-INF  org  weather.jar
[root@mqtt weather]# tree -L 3
.
├── BOOT-INF
│   ├── classes
│   │   ├── adcode.json
│   │   ├── application-dev.yml
│   │   ├── application-pro.yml
│   │   ├── application-test.yml
│   │   ├── application.yml
│   │   ├── caiyun_api.json
│   │   ├── cfg
│   │   ├── com
│   │   ├── db
│   │   ├── log4j.properties
│   │   └── rebel.xml
```

例如添加一个`application-tansuo.yml`文件
然后修改`application.yml`中`spring.profiles.active=tansuo`

## 压缩jar

```bash
jar -cvfM0 weather.jar *
```