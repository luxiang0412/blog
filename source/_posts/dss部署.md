---
title: DSS部署说明
date: 2020-12-08 09:53:04
tags: "dss"
---

## 环境说明

> 软件说明：  

|名称|版本|安装包名称|
|:---|:---|:---|
|JDK|1.8|jdk-8u211-linux-x64.tar.gz|
|Scala|2.11.8|scala-2.11.8.tgz|
|Nginx|1.18|nginx-1.18.0-1.el7.ngx.x86_64.rpm|
|Hadoop|2.7.2|hadoop-2.7.2.tar.gz|
|Hive|1.2.1|apache-hive-1.2.1-bin.tar.gz|
|Spark|2.4.3|spark-2.4.3-bin-2.7.2hive.tgz|
|Centos7.6|7.6|CentOS-7-x86_64-Minimal-1810.iso|
|dss_linkis.zip|`Linkis_0.9.4`,`dss0.9.0`|dss_linkis.zip|
|Mysql|5.7|-|
|Docker|19.03|-|
|docker-compose|-|-|

---

> 服务器说明：

|IP|HOSTNAME|系统版本|
|:---|:---|:---|
|192.168.33.130|centos7test|centos7.6|

修改/etc/hosts
```bash
echo "192.168.33.130    centos7test" >> /etc/hosts

#打开yarn的防火墙
firewall-cmd --permanent --add-port=8088/tcp
#打开dss的防火墙
firewall-cmd --permanent --add-port=9999/tcp
#打开nginx的防火墙
firewall-cmd --permanent --add-port=80/tcp
#刷新规则
firewall-cmd --reload
```

创建hadoop用户
```bash
#添加用户
useradd hadoop
#修改
vi /etc/sudoers
#添加如下规则
hadoop  ALL=(ALL)       NOPASSWD: NOPASSWD: ALL

#切换用户
su hadoop

#生成rsa公钥
ssh-keygen -t rsa -b 4096 -C "centos7test@sd.com"

#公钥写入authorized_keys
echo "$(cat ~/.ssh/id_rsa.pub)" >> ~/.ssh/authorized_keys

eval $(ssh-agent -s)

ssh-add ~/.ssh/id_rsa

chmod 0600 ~/.ssh/authorized_keys
```

---

将以下包上传到`/usr/local`下面

- **jdk-8u211-linux-x64.tar.gz**
- **scala-2.11.8.tgz**
- **hadoop-2.7.2.tar.gz**
- **apache-hive-1.2.1-bin.tar.gz**
- **spark-2.4.3-bin-2.7.2hive.tgz**

将以下包上传到用户目录下（这里我直接上传到`/home/hadoop`下）

- **nginx-1.18.0-1.el7.ngx.x86_64.rpm**
- **dss_linkis.zip**

---

## 配置阿里云镜像源

Base

```bash
#备份
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
#下载阿里云Base Repo
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

#清除缓存
yum clean all

#更新缓存
yum makecache fast
```

EPEL

```bash
#安装epel
yum install epel-release -y

#备份epel repo
mv /etc/yum.repos.d/epel.repo /etc/yum.repos.d/epel.repo.backup
mv /etc/yum.repos.d/epel-testing.repo /etc/yum.repos.d/epel-testing.repo.backup

#下载阿里云epel repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

#清除缓存
yum clean all

#更新缓存
yum makecache fast
```

## Java，Scala，Nginx安装

安装java  

```bash
cd /usr/local

#解压
tar -zxvf jdk-8u211-linux-x64.tar.gz

#创建软连接
ln -s /usr/local/jdk1.8.0_211 /usr/local/java

#配置环境变量
vim /etc/profile

export JAVA_HOME=/usr/local/java
export CLASSPATH=.:$JAVA_HOME/lib:$JRE_HOME/lib
export PATH=$JAVA_HOME/bin:$PATH

#刷新
source /etc/profile

#查看java版本
java -version

```

安装scala  
```bash
# 解压spark包
tar -zxvf scala-2.11.8.tgz

# 创建软链接
ln -s /usr/local/scala-2.11.8 /usr/local/scala

#修改环境变量
vim /etc/profile

export SCALA_HOME=/usr/local/scala
export PATH=$SCALA_HOME/bin:$PATH

#刷新环境变量
source /etc/profile

#查看scala版本
scala -version
```

安装nginx  
```bash
#rpm安装nginx
rpm -ivh nginx-1.18.0-1.el7.ngx.x86_64.rpm

#启动nginx
systemctl start nginx

#开机启动
systemctl enable nginx
```
---

## Docker安装

```bash
#如果有安装docker，先remove
yum remove docker docker-common docker-selinux docker-engine

#安装一些依赖
yum install -y yum-utils device-mapper-persistent-data lvm2

#下载docker repo文件
curl -o /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo

#把软件仓库地址替换为 TUNA
sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo

#生成缓存
yum makecache fast

#查看docker-ce可用版本
yum list --showduplicates docker-ce

#最后安装
yum install docker-ce-19.03.14-3.el7

#配置docker的daemon.json
mkdir /etc/docker 
vim /etc/docker/daemon.json

{
    "registry-mirrors": ["http://hub-mirror.c.163.com"]
}

#启动docker
systemctl start docker

#开机启动docker
systemctl enable docker
```

## Mysql安装和初始化

```bash
#创建mysql目录
mkdir mysql

#下载docker-compose
curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose && \
chmod +x /usr/local/bin/docker-compose

#mysql docker-compose配置
cat > mysql/docker-compose.yml<<'EOF'
version: "3"
services:
    mysql:
        image: mysql:5.7
        container_name: mysql
        ports:
            - "3306:3306"
        volumes:
            - "/mysql_data:/var/lib/mysql"
            - "/mysql_init_data:/docker-entrypoint-initdb.d"
        restart: always
        environment:
            - MYSQL_ROOT_PASSWORD=Csstsari107!
            - TZ=Asia/Shanghai
        command:
            [
                "mysqld",
                "--character-set-server=utf8mb4",
                "--collation-server=utf8mb4_general_ci",
                "--sql_mode="
            ]
EOF

#启动mysql容器
docker-compose up -d

#宿主机安装mysql-client
yum install mysql

#默认密码是Csstsari107
mysql -h 127.0.0.1 -P 3306 -uroot -p

```

创建数据库**dss**和**hive**，后面会用到。（下面是mysql终端执行）
```sql
create database dss default character set utf8mb4 collate utf8mb4_unicode_ci;
create database hive default character set utf8mb4 collate utf8mb4_unicode_ci;
```

## Hadoop,Hive,Spark环境变量配置

```bash
# 环境变量
vim /etc/profile

#hadoop home和config
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop

#hive home和config
export HIVE_HOME=/usr/local/hive
export HIVE_CONF_DIR=$HIVE_HOME/conf

#spark home和config
export SPARK_HOME=/usr/local/spark
export SPARK_CONF_DIR=$SPARK_HOME/conf

export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$SPARK_HOME/bin:$HIVE_HOME/bin:$PATH
```

**请保持Hadoop，Hive，Spark的软链接目录和安装目录权限都是hadoop，包括后边复制的jar包，执行初始化等等**

## 安装Hadoop

```bash
# 解压hadoop包
tar -zxvf hadoop-2.7.2.tar.gz
# 创建软链接
ln -s hadoop-2.7.2 hadoop

```
修改core-site.xml  
`vim hadoop/etc/hadoop/core-site.xml`
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://centos7test:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/local/hadoop/hadoop_data/tmp</value>
    </property>
</configuration>
```

```bash
#修改hadoop-env.sh
vim hadoop/etc/hadoop/hadoop-env.sh

export JAVA_HOME=/usr/local/java
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
```

修改hdfs-site.xml  
`vim hadoop/etc/hadoop/hdfs-site.xml`
```xml
<configuration>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>centos7test:50090</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/usr/local/hadoop/hadoop_data/hdfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/usr/local/hadoop/hadoop_data/hdfs/data</value>
    </property>
</configuration>
```
```bash
#修改mapred-site.xml
cp hadoop/etc/hadoop/mapred-site.xml.template hadoop/etc/hadoop/mapred-site.xml

vim hadoop/etc/hadoop/mapred-site.xml

```
```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>centos7test:10020</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>centos7test:19888</value>
    </property>

</configuration>
```
```bash

#修改slaves
vim hadoop/etc/hadoop/slaves

centos7test


#修改yarn-site.xml
vim hadoop/etc/hadoop/yarn-site.xml

```
```xml
<configuration>
    <property>
        <name>yarn.nodemanager.pmem-check-enabled</name>
        <value>false</value>
    </property>

    <property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
    </property>
    <!-- Site specific YARN configuration properties -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.resourcemanager.address</name>
        <value>centos7test:8032</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>centos7test:8030</value>
    </property>
    <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>centos7test:8031</value>
    </property>
    <property>
        <name>yarn.resourcemanager.admin.address</name>
        <value>centos7test:8033</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>centos7test:8088</value>
    </property>
    <property>
        <name>hadoop.config.diry</name>
        <value>/usr/local/hadoop/etc/hadoop</value>
    </property>
</configuration>
```
```bash
#修改目录所有者
chown -R hadoop:hadoop hadoop hive spark
chown -R hadoop:hadoop hadoop-2.7.2 apache-hive-1.2.1-bin spark-2.4.3-bin-2.7.2hive

su hadoop
#启动
hadoop/bin/hdfs namenode -format

hadoop/sbin/start-dfs.sh
hadoop/sbin/start-yarn.sh
```

## 安装Hive

```bash
# 解压hive包
tar -zxvf apache-hive-1.2.1-bin.tar.gz
# 创建软链接
ln -s apache-hive-1.2.1-bin hive

#修改

cp hive/conf/hive-default.xml.template hive/conf/hive-site.xml
vim hive/conf/hive-site.xml

#替换下面属性
```
```xml
<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://centos7test:3306/hive?createDatabaseIfNotExist=true&amp;characterEncoding=UTF-8&amp;useSSL=false</value>
    <description>JDBC connect string for a JDBC metastore</description>
</property>
<property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
    <description>Driver class name for a JDBC metastore</description>
</property>
<property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
    <description>Username to use against metastore database</description>
</property>
<property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>Csstsari107!</value>
    <description>password to use against metastore database</description>
</property>
```
```bash

#替换变量${system:java.io.tmpdir}和${system:user.name}
:%s#${system:java.io.tmpdir}#/usr/local/hive/tmp#g
:%s#${system:user.name}#hive#g

#复制mysql驱动包到$HIVE_HOME/lib/下面
cp /root/mysql-connector-java-5.1.44.jar $HIVE_HOME/lib/ && \
chown -R hadoop:hadoop $HIVE_HOME/lib/mysql-connector-java-5.1.44.jar

#初始化metastore（前提需要先创建数据库hive）
hive/bin/schematool -dbType mysql -initSchema
```

## 安装Spark

```bash
# 解压spark包
tar -zxvf spark-2.4.3-bin-2.7.2hive.tgz
# 创建软链接
ln -s spark-2.4.3-bin-hadoop2.7 spark

#修改
cp spark/conf/spark-env.sh.template spark/conf/spark-env.sh

vim spark/conf/spark-env.sh

export SPARK_MASTER_IP=centos7test #主机名
export SPARK_WORKER_CORES=2
export SPARK_WORKER_MEMORY=2g
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop

#修改slaves
cp spark/conf/slaves.template spark/conf/slaves
vim spark/conf/slaves
centos7test

#复制mysql驱动到$SPARK_HOME/jars/下
cp $HIVE_HOME/lib/mysql-connector-java-5.1.44.jar $SPARK_HOME/jars/ && \
chown -R hadoop:hadoop $SPARK_HOME/jars/mysql-connector-java-5.1.44.jar

#启动spark-shell
spark/bin/spark-shell --jars $SPARK_HOME/jars/mysql-connector-java-5.1.44.jar
```

## DSS安装

依赖项：

- mysql-client
- nginx
- expect
- dos2unix

```bash
#安装DSS需要的工具
yum install expect dos2unix unzip telnet -y

#解压dss_linkis.zip
unzip -d dss dss_linkis.zip

#进入安装目录
cd dss

#修改数据配置文件
vim conf/db.sh

MYSQL_HOST=centos7test
MYSQL_PORT=3306
MYSQL_DB=dss
MYSQL_USER=root
MYSQL_PASSWORD=Csstsari107!

#修改安装配置文件
cp conf/config.sh.standard.template conf/config.sh
vim conf/config.sh
```
文件`config.sh`

```bash
##########################################################################################
#######################  Must provided Configurations  ###################################
##########################################################################################

### The install home path of Linkis

#!/bin/sh
#Actively load user env

SSH_PORT=22

deployUser="`whoami`"

### Specifies the user workspace, which is used to store the user's script files and log files.
### Generally local directory
WORKSPACE_USER_ROOT_PATH=file:///home/hadoop/linkis/ ##file:// required

### User's root hdfs path
HDFS_USER_ROOT_PATH=hdfs:///home/hadoop/linkis ##hdfs:// required

### Path to store job ResultSet:file or hdfs path
RESULT_SET_ROOT_PATH=hdfs:///home/hadoop/linkis

# Used to store the azkaban project transformed by DSS
WDS_SCHEDULER_PATH=file:///home/hadoop/wds/scheduler

#DSS Web
DSS_NGINX_IP=127.0.0.1
DSS_WEB_PORT=9999

###azkaban  address for check
AZKABAN_ADRESS_IP=127.0.0.1
AZKABAN_ADRESS_PORT=8081
#
####qualitis address for check
QUALITIS_ADRESS_IP=127.0.0.1
QUALITIS_ADRESS_PORT=8090

##hive metastore
HIVE_META_URL=jdbc:mysql://centos7test:3306/hive?useUnicode=true
HIVE_META_USER=root
HIVE_META_PASSWORD=Csstsari107!

###HADOOP CONF DIR
HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop

###HIVE CONF DIR
HIVE_CONF_DIR=/usr/local/hive/conf

###SPARK CONF DIR
SPARK_CONF_DIR=/usr/local/spark/conf


##########################################################################################
####The following parameters can be modified by the user as required, but not necessary###
##########################################################################################

## LDAP is for enterprise authorization, if you just want to have a try, ignore it.
#LDAP_URL=ldap://localhost:1389/
#LDAP_BASEDN=dc=webank,dc=com

## java application default jvm memory
export SERVER_HEAP_SIZE="512M"

LINKIS_VERSION=0.9.4

DSS_VERSION=0.9.0
```

将引擎的启动参数调整一下
例如：
`linkis/linkis-ujes-xxx-enginemanager`文件下的`conf/linkis.properties`

```bash
#用于指定xxxEM启动的所有引擎的客户端的总内存
wds.linkis.enginemanager.memory.max=8G
#用于指定xxxEM启动的所有引擎的客户端的总CPU核数
wds.linkis.enginemanager.cores.max=8
#用于指定xxxEM可以启动的引擎个数
wds.linkis.enginemanager.engine.instances.max=8
```

## DSS启动

```
sh bin/start-all.sh
```
## FAQ
1.nginx访问出现403，将/etc/nginx/nginx.conf中的user改为hadoop  

2.数据库刷不出来，hive终端授权`grant all on database default to user hadoop;`

3.获取Yarn队列信息异常将linkis-resourcemanager-server-0.9.4.jar包替换一下


