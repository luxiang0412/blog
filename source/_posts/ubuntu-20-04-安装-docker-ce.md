---
title: ubuntu 20.04 安装 docker-ce
date: 2021-01-18 08:46:12
tags:
---


## 安装

国内服务器替换url为`https://mirrors.aliyun.com/docker-ce/linux/ubuntu`
```bash
sudo apt update && \
sudo apt install apt-transport-https ca-certificates curl software-properties-common && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - && \
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable" && \
sudo apt update && \
apt-cache policy docker-ce && \
sudo apt install docker-ce
```

## 参考

- [how-to-install-and-use-docker-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04)