---
title: ubuntu20.04 server init
date: 2021-01-09 22:51:43
tags:
---

## 允许root登录
```bash
sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

sudo systemctl restart ssh
```

## 注入公钥
```bash
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDKOcVX5KCpO43id56L3PW0pyYBlWasorrsIMChYL1Zl+wUnOX3YoTcjNyFZwD62bzasBxOHtPdrPQ0CHAvjo78PG8De01MqcFRo6RNnKbqr4Zq/hfIJWyBs7PmHFuXKpI47Ieoy5gfHdDP+RaqWAalUHa7gy14R5zPh9gvWf3s8pL5/Wbk4lPj+Kdrq2JOUU0fclc/WjvroIAbpp1ofnO+1EBMBgcs6F58WyFJBtgWXgAstK8QPb5tI1OpJhwLf0nVyA5M8kbkHiu1mZiEyqRqMJbGNhip7wYU1yy5+ad2nhL7cqmENJlFgVKdxoDTTxJfYf+cIgqrkDgiKRdA271279HMChSaCWo7XNadk/0aDRp/XenHFDQHw4YEiND8TkY/JpL0YjtmIFepIjcol5you5DnbqanLCzaLnrzCqrs+Wkjr+vEr8EZ59VPrMemaPSJ1DkvgzXDGuGry+7UcDRZNg+c2ocerJ3aQPNdq9L1iD9YO1ATYZH/bg2sdOpvfjop5jh1PB98AYhYy6YTANISgSYVk4D/DMSg9DKyOKkpZGNQQvamRR6ODXakhcWzvNZMDYlCE6isma1/xxRMdPsXz5ZgNJi5hX7Olg1pQe3UYSFZqK7DMi2R3Nwm96zP39EH6YNDG3UcnXxdi65ACHabhhjCfUela9VNwHS+K4z4UQ== luxiang19960412@gmail.com" >> /root/.ssh/authorized_keys
```

## apt源修改
```bash
替换/etc/apt/sources.list地址
```

## 更新软件包
```bash
apt update -y && apt upgrade -y
```

## 安装docker-ce
```bash
apt update && \
apt install -y \
apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

apt update && \
apt install docker-ce -y
```

docker registry mirrors
```bash
cat > /etc/docker/daemon.json <<EOF
{
    "registry-mirrors": [
        "http://hub-mirror.c.163.com",
        "https://registry.cn-hangzhou.aliyuncs.com",
        "https://ustc-edu-cn.mirror.aliyuncs.com",
        "https://dockerhub.azk8s.cn",
        "https://registry.cn-hangzhou.aliyuncs.com"
    ]
}
```
启动docker-ce
```bash
systemctl start docker && \
systemctl enable docker
```

## 安装常用的软件
```bash
apt install net-tools bind9-utils nmap unzip zip -y
```

## 网络配置

`/etc/netplan/00-installer-config.yaml`
```bash
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.36.135/24
      gateway4: 192.168.36.2
      nameservers:
        addresses: [114.114.114.114, 8.8.8.8]
  version: 2
```
