---
title: golang之远程开发
date: 2021-01-20 22:32:38
tags:
---

## 升级服务端的git至2.x

```bash
sudo yum -y install https://packages.endpoint.com/rhel/7/os/x86_64/endpoint-repo-1.7-1.x86_64.rpm
sudo yum install git
git --version
```

## 创建一个普通用户luxiang

```bash
useradd luxiang
echo "luxiang  ALL=(ALL)       NOPASSWD: NOPASSWD: ALL">> /etc/sudoers
```

## 将自己的公钥写入authorized_keys

```bash
mkdir .ssh
chmod 700 .ssh
echo "$PUBLICH_KEY" >> .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
```

## 下载golang

```bash
mkdir local
cd local
wget https://golang.google.cn/dl/go1.15.7.linux-amd64.tar.gz
tar -zxvf go1.15.7.linux-amd64.tar.gz
# 配置go环境变量和镜像地址
cat >> ~/.bash_profile << 'EOF'
export GOPATH=/home/luxiang/local/go
export PATH=$PATH:$GOPATH/bin
export GO111MODULE=on
export GOPROXY=https://goproxy.cn
EOF

go env
```
## 初始话一个项目

```bash
mkdir gotest
cd gotest
go mod init gitee.com/luxiang0412/gotest

cat go.mod
```
```bash
cat > main.go << 'EOF'
```
```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, world.")
}
```

```bash
go install gitee.com/luxiang0412/gotest

git init
git remote add origin git@gitee.com:luxiang0412/gotest.git
git add --all
git config --global user.email "1374920889@qq.com"
git config --global user.name "luxiang"
git commit -m 'init'
git push -u origin master

go get github.com/beego/bee

```

## vscode 安装 remote 插件

## vscode add host

## 参考

- [vscode+go远程开发](https://zhuanlan.zhihu.com/p/73799399)
- [Install Latest Git ( Git 2.x ) on CentOS 7](https://computingforgeeks.com/how-to-install-latest-version-of-git-git-2-x-on-centos-7/)
- [golang](https://golang.google.cn)
- [goproxy](https://goproxy.cn)
- [remote-development](https://code.visualstudio.com/blogs/2019/05/02/remote-development)
- [troubleshooting](https://code.visualstudio.com/docs/remote/troubleshooting)