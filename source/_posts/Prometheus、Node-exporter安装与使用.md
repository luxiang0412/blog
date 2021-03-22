---
title: Prometheus、Node exporter安装与使用
date: 2021-03-22 17:26:22
tags:
---


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [环境](#环境)
- [安装Docker](#安装docker)
- [安装Prometheus](#安装prometheus)
- [安装Node exporter](#安装node-exporter)
- [使用](#使用)
- [参考](#参考)

<!-- /code_chunk_output -->

Prometheus 用于抓取或者接收Exporter采集的监控指标

架构图如下：

<center>
    <img src="architecture.png" width="100%" height="100%">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Prometheus监控体系架构图</div>
</center>

## 环境

- `Centos7.6`
- `Docker CE 1.9`

## 安装Docker

安装Docker
```bash
yum install -y yum-utils device-mapper-persistent-data lvm2 && \
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo && \
yum makecache fast && \
yum -y install docker-ce
```

启动Docker
```bash
systemctl start docker && \
systemctl enable docker
```

## 安装Prometheus

```bash
# pull Prometheus
docker pull prom/prometheus

# 启动Prometheus
docker run -d --name prometheus -p 9090:9090 --restart=always -v ./prometheus.yml:/etc/prometheus/prometheus.yml:ro prom/prometheus -config.file=/etc/prometheus/prometheus.yml
```

`prometheus.yml`配置文件如下，这里假设采集`192.168.1.11`和`192.168.1.13`两台机器
```yaml
scrape_configs:
  - job_name: nodeexporter
    scrape_interval: 5s
    static_configs:
      - targets: ["192.168.1.11:9100"]
        labels:
          ip_address: "192.168.1.11"
      - targets: ["192.168.1.13:9100"]
        labels:
          ip_address: "192.168.1.13"
```

## 安装Node exporter

```bash
# 拉取镜像
docker pull prom/node-exporter

# 启动Node exporter
docker run -d --name node-exporter -v /proc:/host/proc:ro -v /sys:/host/sys:ro -v /:/rootfs:ro --network host -p 9100:9100 --restart=always prom/node-exporter '--path.procfs=/host/proc' '--path.sysfs=/host/sys' '--collector.filesystem.ignored-mount-points' '^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)'
```

## 使用

Prometheus使用

浏览器访问：[http://127.0.0.1:9090](http://127.0.0.1:9090) 

如图：

<center>
    <img src="prometheus.jpg" width="100%" height="100%">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Prometheus界面</div>
</center>

Node-exporter使用

浏览器访问：[http://127.0.0.1:9100/metrics](http://127.0.0.1:9100/metrics)

如图：
<center>
    <img src="node-exporter.jpg" width="100%" height="100%">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Node-exporter metrics</div>
</center>

## 参考

- [Prometheus installation](https://prometheus.io/docs/prometheus/latest/installation)
- [Node-exporter](https://github.com/prometheus/node_exporter)