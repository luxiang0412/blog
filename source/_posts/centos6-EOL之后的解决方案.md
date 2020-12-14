---
title: centos6 EOL之后的解决方案
date: 2020-12-14 11:59:11
tags: 
    - "centos6"
    - "epel6"
    - "yum"
---

官方已经不再维护centos6 的源。处理方案如下：

### 使用CentOS Vault源

```bash
vi /etc/yum/pluginconf.d/fastestmirror.conf
#修改参数
enable=0
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
curl -o /etc/yum.repos.d/CentOS-Base.repo https://static.lty.fun/%E5%85%B6%E4%BB%96%E8%B5%84%E6%BA%90/SourcesList/Centos-6-Vault-Aliyun.repo
```

### 修复EPEL源
```bash
curl https://www.getpagespeed.com/files/centos6-epel-eol.repo --output /etc/yum.repos.d/epel.repo
```
### 修复SCLO源

```bash
yum -y install centos-release-scl
curl https://www.getpagespeed.com/files/centos6-scl-eol.repo --output /etc/yum.repos.d/CentOS-SCLo-scl.repo
curl https://www.getpagespeed.com/files/centos6-scl-rh-eol.repo --output /etc/yum.repos.d/CentOS-SCLo-scl-rh.repo
```

### 清理与缓存

```bash
yum clean all
yum makecache
```

### 参考

- [centos6 yum源不能使用](https://developer.aliyun.com/article/779734)
- [再见！Centos6](https://segmentfault.com/a/1190000038396431)
- [how-to-fix-yum-after-centos-6-went-eol](https://www.getpagespeed.com/server-setup/how-to-fix-yum-after-centos-6-went-eol)