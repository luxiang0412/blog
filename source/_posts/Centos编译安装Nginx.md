---
title: Centos编译安装Nginx
date: 2021-05-28 16:59:52
tags:
---

```bash
# 检查依赖
rpm -qa|egrep 'openssl-devel|pcre|zlib'
```

    pcre-7.8-7.el6.x86_64
    zlib-devel-1.2.3-29.el6.x86_64
    openssl-devel-1.0.1e-58.el6_10.x86_64
    pcre-devel-7.8-7.el6.x86_64
    zlib-1.2.3-29.el6.x86_64

安装依赖
```bash
yum install openssl-devel gcc pcre-devel -y
```

```bash
mkdir -p /web/nginx && \
mkdir /web/nginx/modules && \
mkdir  /web/nginx/run && \
cd /web/nginx/ && \
cp /root/nginx-1.20.1.tar.gz /web/nginx/ && \
tar -zxvf nginx-1.20.1.tar.gz && \
mkdir binaries && \
mv nginx-1.20.1/* binaries && \
rm  -rf nginx-1.20.1/ && \
cd binaries/ && \
./configure --prefix=/web/nginx --modules-path=/web/nginx/modules --with-http_ssl_module  --without-http_fastcgi_module --without-http_uwsgi_module --without-http_grpc_module --without-http_scgi_module --without-mail_imap_module --without-mail_pop3_module


# make
make

# make install
make install

cd /web/nginx/ && \
rm  -rf binaries/

# 启动
/web/nginx/sbin/nginx  -c  /web/nginx/conf/nginx.conf

# 停止
pkill -F /web/nginx/logs/nginx.pid
```