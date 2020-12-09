---
title: dss部署
date: 2020-12-08 09:53:04
tags: dss
---

# dss部署

安装包

- scala-2.11.8.tgz
- hadoop-2.7.2.tar.gz
- apache-hive-1.2.1-bin.tar.gz
- spark-2.4.3-bin-hadoop2.7.tgz
- dss_linkis.zip
- mysql-connector-java-5.1.44.jar

将**hadoop-2.7.2.tar.gz**,**apache-hive-1.2.1-bin.tar.gz**,**spark-2.4.3-bin-hadoop2.7.tgz**包上传到`/usr/local`下面

## 0.hadoop,hive,spark环境变量配置

```bash
# 环境变量
vim /etc/profile

export SCALA_HOME=/usr/local/scala
export HADOOP_HOME=/usr/local/hadoop
export SPARK_HOME=/usr/local/spark
export HIVE_HOME=/usr/local/hive
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$SPARK_HOME/bin:$HIVE_HOME/bin:$SCALA_HOME/bin:$PATH
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HIVE_CONF_DIR=$HIVE_HOME/conf
export SPARK_CONF_DIR=$SPARK_HOME/conf
```

## 3.安装scala

```bash
# 解压spark包
tar -zxvf scala-2.11.8.tgz
# 创建软链接
ln -s scala-2.11.8 scala
```

## 1.安装hadoop

```bash
# 解压hadoop包
tar -zxvf hadoop-2.7.2.tar.gz
# 创建软链接
ln -s hadoop-2.7.2 hadoop

#修改core-site.xml
vim hadoop/etc/hadoop/core-site.xml

<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://10.7.200.209:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/local/hadoop/hadoop_data/tmp</value>
    </property>
</configuration>

#修改hadoop-env.sh
vim hadoop/etc/hadoop/hadoop-env.sh

export JAVA_HOME=/opt/jdk1.8.0_211
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop

#修改hdfs-site.xml
vim hadoop/etc/hadoop/hdfs-site.xml

<configuration>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>10.7.200.209:50090</value>
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

#修改mapred-site.xml
cp hadoop/etc/hadoop/mapred-site.xml.template hadoop/etc/hadoop/mapred-site.xml

vim hadoop/etc/hadoop/mapred-site.xml

<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>10.7.200.209:10020</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>10.7.200.209:19888</value>
    </property>

</configuration>

#修改slaves
vim hadoop/etc/hadoop/slaves

10.7.200.209


#修改yarn-site.xml
vim hadoop/etc/hadoop/yarn-site.xml

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
        <value>10.7.200.209:8032</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>10.7.200.209:8030</value>
    </property>
    <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>10.7.200.209:8031</value>
    </property>
    <property>
        <name>yarn.resourcemanager.admin.address</name>
        <value>10.7.200.209:8033</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>10.7.200.209:8088</value>
    </property>
    <property>
        <name>hadoop.config.diry</name>
        <value>/usr/local/hadoop/etc/hadoop</value>
    </property>
</configuration>


#生成ssh密钥
ssh-keygen -b 4096 -t rsa
cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys

#启动
hadoop/bin/hdfs namenode -format

hadoop/sbin/start-dfs.sh
hadoop/sbin/start-yarn.sh
```

## 2.安装hive

```bash
# 解压hive包
tar -zxvf apache-hive-1.2.1-bin.tar.gz
# 创建软链接
ln -s apache-hive-1.2.1-bin hive

#修改

cp hive/conf/hive-default.xml.template hive/conf/hive-site.xml
vim hive/conf/hive-site.xml

#替换下面属性
<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://10.7.200.209:3306/hive?createDatabaseIfNotExist=true&amp;characterEncoding=UTF-8&amp;useSSL=false</value>
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

#替换变量${system:java.io.tmpdir}和${system:user.name}
:%s#${system:java.io.tmpdir}#/usr/local/hive/tmp#g
:%s#${system:user.name}#hive#g

#复制mysql驱动包到$HIVE_HOME/lib/下面
cp /root/mysql-connector-java-5.1.44.jar $HIVE_HOME/lib/

#初始化metastore（前提需要先创建数据库hive）
hive/bin/schematool -dbType mysql -initSchema
```

## 3.安装spark

```bash
# 解压spark包
tar -zxvf spark-2.4.3-bin-hadoop2.7.tgz
# 创建软链接
ln -s spark-2.4.3-bin-hadoop2.7 spark

#修改
cp spark/conf/spark-env.sh.template spark/conf/spark-env.sh

vim spark/conf/spark-env.sh

export SPARK_MASTER_IP=10.7.200.209 #主机名
export SPARK_WORKER_CORES=2
export SPARK_WORKER_MEMORY=2g
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop

#修改slaves
cp spark/conf/slaves.template spark/conf/slaves
vim spark/conf/slaves
10.7.200.209

#复制mysql驱动到$SPARK_HOME/jars/下
cp $HIVE_HOME/lib/mysql-connector-java-5.1.44.jar $SPARK_HOME/jars/

#启动spark-shell
spark/bin/spark-shell --jars $SPARK_HOME/jars/mysql-connector-java-5.1.44.jar
```

## 4.dss安装

依赖项：

- mysql-client
- nginx
- expect
- dos2unix

```bash
#安装dss需要的工具
yum install 

#解压dss_linkis.zip
unzip -d dss dss_linkis.zip

#进入安装目录
cd dss

#修改数据配置文件
vim conf/db.sh

MYSQL_HOST=10.7.200.209
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
WORKSPACE_USER_ROOT_PATH=file:///tmp/linkis/ ##file:// required

### User's root hdfs path
HDFS_USER_ROOT_PATH=hdfs:///tmp/linkis ##hdfs:// required

### Path to store job ResultSet:file or hdfs path
RESULT_SET_ROOT_PATH=hdfs:///tmp/linkis

# Used to store the azkaban project transformed by DSS
WDS_SCHEDULER_PATH=file:///appcom/tmp/wds/scheduler

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
HIVE_META_URL=jdbc:mysql://10.7.200.209:3306/hive?useUnicode=true
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
wds.linkis.enginemanager.cores.max=4
#用于指定xxxEM可以启动的引擎个数
wds.linkis.enginemanager.engine.instances.max=4
```

## 5.dss启动

```
sh bin/start-all.sh
```
