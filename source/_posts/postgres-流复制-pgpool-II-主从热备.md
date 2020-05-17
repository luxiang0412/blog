---
title: postgres 流复制 + pgpool-II 主从热备
date: 2020-05-17 15:06:37
tags: postgres
categories: postgres
---
### 环境准备
- CentOS release 6.10
- postgresql-9.4
- pgpool-II 4.1.1
- 准备两台机器
    pmotest1    192.168.64.129  作为master主节点
    pmotest2    192.168.64.130  作为slave从节点
- 一个虚拟ip
    vip         192.168.64.20

### 安装postgres
```
yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-6-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install postgresql94
yum install postgresql94-server
service postgresql-9.4 initdb
service postgresql-9.4 start
chkconfig postgresql-9.4 on
```

### master节点初始化，配置
修改postgres密码，并添加repuser 用于流复制
```
postgres=# ALTER USER postgres WITH PASSWORD 'postgres';
postgres=# CREATE ROLE repuser WITH REPLICATION PASSWORD 'repuser' LOGIN;
```
修改pg_hba.conf文件（密码认证）
```
local   all             all                                     md5
# IPv4 local connections:
host    all             all             0.0.0.0/0               md5
# IPv6 local connections:
host    all             all             0/0                     md5
host    replication     repuser         0.0.0.0/0               md5
```
修改postgres.conf配置文件
```
listen_addresses = '*'
max_connections = 500
password_encryption = on
wal_level = hot_standby
archive_mode = on
archive_command = 'cp %p /var/lib/pgsql/9.4/pg_archive/%f'
max_wal_senders = 5
wal_keep_segments = 32
hot_standby = on
```
添加所有节点的环境变量PGHOME PGDATA
```
$ vim ~/.bash_profile

PGDATA=/var/lib/pgsql/9.4/data
PGHOME=/usr/pgsql-9.4
PATH=$PATH:$PGHOME/bin
export PGDATA
export PGHOME
export PATH

$ source ~/.bash_profile
```
    在maser 重新启动postgres数据库
    在slave节点上，删除$PGDATA文件
```
#同步主节点数据文件
$ /usr/bin/pg_basebackup -h 192.168.64.129 -D $PGDATA -P -U repuser --xlog-method=stream
#创建recovery.conf文件

standby_mode = on
primary_conninfo = 'host=192.168.64.129 port=5432 user=repuser password=repuser'
trigger_file = '/var/lib/pgsql/9.4/tigger'
recovery_target_timeline = 'latest'

```

    测试热备，在slave节点上执行
```
$ $PGHOME/bin/pg_ctl promote -D $PGDATA
```
    此时slave升级为主节点，现在测试将原来的master节点加入到slave
    创建recovery.conf文件，与上面的类似，只是ip地址变了
### pgpool-II功能
- 连接池
- 负载均衡
- 自动故障转移
- 在线恢复
- 数据同步
- 最大连接数限制
- Watchdog
- 内存缓存查询

### pgpool-II安装
```
$ yum install https://www.pgpool.net/yum/rpms/4.1/redhat/rhel-6-x86_64/pgpool-II-release-4.1-1.noarch.rpm
$ yum install pgpool-II-pg94
$ yum install pgpool-II-pg94-debuginfo
$ yum install pgpool-II-pg94-devel
$ yum install pgpool-II-pg94-extensions
$ pgpool --version
pgpool-II version 4.1.1 (karasukiboshi)
$ psql -V
psql (PostgreSQL) 9.4.24
```

### pgpool-II配置
修改pool_hba.conf
```
local   all             all                                     md5
# IPv4 local connections:
host    all             all             0.0.0.0/0               md5
# IPv6 local connections:
host    all             all             0/0                     md5
```
配置pcp.conf
```
postgres:postgres
```
生成pool_passwd MD5
```
$ pg_md5 -p -m -u postgres pool_passwd
```
配置系统命令权限
```
$ which ip
/sbin/ip
$ chmod u+s /sbin/ip
$ chmod u+s /sbin
$ chmod u+s /usr/sbin/arping
```
配置
#### 提示

    提示
    先启动pg数据库，然后再启动pgpool
    先停止pgpool，再停止pg数据库

### 配置

### 参考
- [postgres install](https://www.postgresql.org/download/linux/redhat/)
- [Streaming_Replication](https://wiki.postgresql.org/wiki/Streaming_Replication)
- [pgpool Wiki](https://www.pgpool.net/mediawiki/index.php/Main_Page)
- [pgpool-II 4.1.1 document](https://www.pgpool.net/docs/latest/en/html/index.html)
- [PGPool-II+PG流复制实现HA主备切换](https://www.jianshu.com/p/ef183d0a9213)