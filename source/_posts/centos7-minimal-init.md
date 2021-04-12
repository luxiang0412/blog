---
title: centos7 minimal init
date: 2021-01-09 17:47:27
tags: centos7
---

## Base、epel、docker安装或替换


```bash
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo && \
sed -i 's#http#https#g' /etc/yum.repos.d/epel.repo
```

Base镜像替换
```bash
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup && \
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo && \
sed -i 's#http#https#g' /etc/yum.repos.d/CentOS-Base.repo
```

```bash
yum clean all && yum makecache && yum -y update
```

docker-ce
```bash
yum install -y yum-utils device-mapper-persistent-data lvm2 && \
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo && \
yum makecache fast && \
yum -y install docker-ce
```

docker daemon.json
```
mkdir /etc/docker && cat > /etc/docker/daemon.json <<EOF
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

```bash
systemctl start docker && \
systemctl enable docker
```

## 公钥注入

```bash
mkdir ~/.ssh && echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDKOcVX5KCpO43id56L3PW0pyYBlWasorrsIMChYL1Zl+wUnOX3YoTcjNyFZwD62bzasBxOHtPdrPQ0CHAvjo78PG8De01MqcFRo6RNnKbqr4Zq/hfIJWyBs7PmHFuXKpI47Ieoy5gfHdDP+RaqWAalUHa7gy14R5zPh9gvWf3s8pL5/Wbk4lPj+Kdrq2JOUU0fclc/WjvroIAbpp1ofnO+1EBMBgcs6F58WyFJBtgWXgAstK8QPb5tI1OpJhwLf0nVyA5M8kbkHiu1mZiEyqRqMJbGNhip7wYU1yy5+ad2nhL7cqmENJlFgVKdxoDTTxJfYf+cIgqrkDgiKRdA271279HMChSaCWo7XNadk/0aDRp/XenHFDQHw4YEiND8TkY/JpL0YjtmIFepIjcol5you5DnbqanLCzaLnrzCqrs+Wkjr+vEr8EZ59VPrMemaPSJ1DkvgzXDGuGry+7UcDRZNg+c2ocerJ3aQPNdq9L1iD9YO1ATYZH/bg2sdOpvfjop5jh1PB98AYhYy6YTANISgSYVk4D/DMSg9DKyOKkpZGNQQvamRR6ODXakhcWzvNZMDYlCE6isma1/xxRMdPsXz5ZgNJi5hX7Olg1pQe3UYSFZqK7DMi2R3Nwm96zP39EH6YNDG3UcnXxdi65ACHabhhjCfUela9VNwHS+K4z4UQ== luxiang19960412@gmail.com" >> ~/.ssh/authorized_keys
```

## 常用软件安装
```bash
yum install vim bind-utils net-tools wget telnet nmap gcc unzip zip -y
```
## 网络配置


```bash
sed -i '/^BOOTPROTO/cBOOTPROTO="static"' /etc/sysconfig/network-scripts/ifcfg-ens33 && \
sed -i '/^ONBOOT/cONBOOT="yes"' /etc/sysconfig/network-scripts/ifcfg-ens33 && \
sed -i '$aIPADDR=192.168.36.133' /etc/sysconfig/network-scripts/ifcfg-ens33 && \
sed -i '$aNETMASK=255.255.255.0' /etc/sysconfig/network-scripts/ifcfg-ens33 && \
sed -i '$aGATEWAY=192.168.36.2' /etc/sysconfig/network-scripts/ifcfg-ens33 && \
sed -i '$aDNS1=114.114.114.114' /etc/sysconfig/network-scripts/ifcfg-ens33 && \
sed -i '$aDNS2=8.8.8.8' /etc/sysconfig/network-scripts/ifcfg-ens33
```

```bash
systemctl restart network
```
