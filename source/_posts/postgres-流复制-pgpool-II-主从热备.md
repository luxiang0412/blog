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
- 准备四台机器，作用分别如下
    pmotest1    192.168.64.129  装postgres和pgpool
    pmotest2    192.168.64.130  装postgres和pgpool
    pmotest3    192.168.64.131  装pgpool
    pmotest4    192.168.64.128  装pgpool
- 一个虚拟ip
    vip         192.168.64.20

### 在pmotest1和pmotest2安装postgres
```
yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-6-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install postgresql94
yum install postgresql94-server
```

### master节点初始化，配置
初始化pmotest1的数据库文件，并启动pmotest1数据库
```
$ /etc/init.d/postgresql-9.4 initdb
$ /etc/init.d/postgresql-9.4 start
$ chkconfig postgresql-9.4 on
```
修改postgres密码，并添加repuser 用于流复制
```
$ psql
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
#监听
listen_addresses = '*'
#连接数
max_connections = 500
password_encryption = on
wal_level = hot_standby
#归档
archive_mode = on
#归档命令
archive_command = 'cp %p /var/lib/pgsql/9.4/pg_archive/%f'
max_wal_senders = 5
wal_keep_segments = 32
hot_standby = on
```
添加所有节点的环境变量PGHOME PGDATA
```
$ vim /etc/profile

export PGDATA=/var/lib/pgsql/9.4/data
export PGHOME=/usr/pgsql-9.4
export PATH=$PATH:$PGHOME/bin

$ source /etc/profile
```
重新启动pmotest1的postgres（到此pmotest1的postgres配置完毕）
```
/etc/init.d/postgresql-9.4 restart
```
开始pmotest2的postgres配置，首先复制pmotest1的复制文件，命令如下（注意环境变量$PGDATA是否存在）
```
$ /usr/bin/pg_basebackup -h pmotest1 -D $PGDATA -P -U repuser --xlog-method=stream
```
配置recovery.conf文件
```
#创建recovery.conf文件

standby_mode = on
primary_conninfo = 'host=192.168.64.129 port=5432 user=repuser password=repuser'
trigger_file = '/var/lib/pgsql/9.4/tigger'
recovery_target_timeline = 'latest'

```
pmotest2的postgres配置完毕

启动pmotest2的pg
```
$ pg_ctl start
```
测试主备切换
```
#模拟master宕机（因为此时pmotest1是master，所以在pmotest1上执行如下命令）
$ pg_ctl stop -m fast
#开始模拟slave节点 升级为master节点
$ $PGHOME/bin/pg_ctl promote -D $PGDATA
```
此时slave升级为主节点，现在将原来的master节点加入到集群
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
配置系统命令权限和文件夹
```
$ which ip
/sbin/ip
$ chmod u+s /sbin/ip
$ chmod u+s /sbin
$ chmod u+s /usr/sbin/arping
$ mkdir /var/log/pgpool /var/run/pgpool/
$ chown -R postgres:postgres /var/log/pgpool
$ chown -R postgres:postgres /var/run/pgpool/
```
pgpool.conf配置
```
#监听
listen_addresses = '*'
#pgpool 对外连接的端口  也就是java 程序中需要填写的端口
port = 9999
#socket 路径
socket_dir = '/tmp'
listen_backlog_multiplier = 2
                                   # Set the backlog parameter of listen(2) to
                                   # num_init_children * listen_backlog_multiplier.
                                   # (change requires restart)
serialize_accept = off
                                   # whether to serialize accept() call to avoid thundering herd problem
                                   # (change requires restart)
reserved_connections = 0
                                   # Number of reserved connections.
                                   # Pgpool-II does not accept connections if over
                                   # num_init_chidlren - reserved_connections.
pcp_listen_addresses = '*'
                                   # Host name or IP address for pcp process to listen on:
                                   # '*' for all, '' for no TCP/IP connections
                                   # (change requires restart)
pcp_port = 9898
                                   # Port number for pcp
                                   # (change requires restart)
pcp_socket_dir = '/tmp'
                                   # Unix domain socket path for pcp
                                   # The Debian package defaults to
                                   # /var/run/postgresql
                                   # (change requires restart)
#后端第一个postgres的hostname
backend_hostname0 = 'pmotest1'
#后端第一个postgres的端口
backend_port0 = 5432
#后端第一个postgres的权重
backend_weight0 = 1
#后端第一个postgres的 PGDATA 数据目录
backend_data_directory0 = '/var/lib/pgsql/9.4/data'
#后端第一个postgres的名称 通过命令show pool_nodes  展示的名称
backend_application_name0 = 'pmotest1'
backend_flag0 = 'ALLOW_TO_FAILOVER'
                                   # Controls various backend behavior
                                   # ALLOW_TO_FAILOVER, DISALLOW_TO_FAILOVER
                                   # or ALWAYS_MASTER
#同理第一个个后端postgres配置      
backend_hostname1 = 'pmotest2'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/var/lib/pgsql/9.4/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'
backend_application_name1 = 'pmotest2'
enable_pool_hba = on
                                   # Use pool_hba.conf for client authentication
pool_passwd = 'pool_passwd'
                                   # File name of pool_passwd for md5 authentication.
                                   # "" disables pool_passwd.
                                   # (change requires restart)
authentication_timeout = 60
                                   # Delay in seconds to complete client authentication
                                   # 0 means no timeout.
allow_clear_text_frontend_auth = off
                                   # Allow Pgpool-II to use clear text password authentication
                                   # with clients, when pool_passwd does not
                                   # contain the user password
ssl = off
                                   # Enable SSL support
                                   # (change requires restart)
                                   # Path to the SSL private key file
                                   # (change requires restart)
                                   # Path to the SSL public certificate file
                                   # (change requires restart)
                                   # Path to a single PEM format file
                                   # containing CA root certificate(s)
                                   # (change requires restart)
                                   # Directory containing CA root certificate(s)
                                   # (change requires restart)
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'
                                   # Allowed SSL ciphers
                                   # (change requires restart)
ssl_prefer_server_ciphers = off
                                   # Use server's SSL cipher preferences,
                                   # rather than the client's
                                   # (change requires restart)
ssl_ecdh_curve = 'prime256v1'
                                   # Name of the curve to use in ECDH key exchange
ssl_dh_params_file = ''
                                   # Name of the file containing Diffie-Hellman parameters used
                                   # for so-called ephemeral DH family of SSL cipher.
num_init_children = 32
                                   # Number of concurrent sessions allowed
                                   # (change requires restart)
max_pool = 4
                                   # Number of connection pool caches per connection
                                   # (change requires restart)
child_life_time = 300
                                   # Pool exits after being idle for this many seconds
child_max_connections = 0
                                   # Pool exits after receiving that many connections
                                   # 0 means no exit
connection_life_time = 0
                                   # Connection to backend closes after being idle for this many seconds
                                   # 0 means no close
client_idle_limit = 0
                                   # Client is disconnected after being idle for that many seconds
                                   # (even inside an explicit transactions!)
                                   # 0 means no disconnection
log_destination = 'stderr'
                                   # Where to log
                                   # Valid values are combinations of stderr,
                                   # and syslog. Default to stderr.
log_line_prefix = '%t: pid %p: '   # printf-style string to output at beginning of each log line.
log_connections = off
                                   # Log connections
log_hostname = off
                                   # Hostname will be shown in ps status
                                   # and in logs if connections are logged
log_statement = off
                                   # Log all statements
log_per_node_statement = off
                                   # Log all statements
                                   # with node and backend informations
log_client_messages = off
                                   # Log any client messages
log_standby_delay = 'none'
                                   # Log standby delay
                                   # Valid values are combinations of always,
                                   # if_over_threshold, none
syslog_facility = 'LOCAL0'
                                   # Syslog local facility. Default to LOCAL0
syslog_ident = 'pgpool'
                                   # Syslog program identification string
                                   # Default to 'pgpool'
                                        #   debug5
                                        #   debug4
                                        #   debug3
                                        #   debug2
                                        #   debug1
                                        #   log
                                        #   notice
                                        #   warning
                                        #   error
                                        #   debug5
                                        #   debug4
                                        #   debug3
                                        #   debug2
                                        #   debug1
                                        #   info
                                        #   notice
                                        #   warning
                                        #   error
                                        #   log
                                        #   fatal
                                        #   panic
pid_file_name = '/var/run/pgpool/pgpool.pid'
                                   # PID file name
                                   # Can be specified as relative to the"
                                   # location of pgpool.conf file or
                                   # as an absolute path
                                   # (change requires restart)
logdir = '/var/log/pgpool'
                                   # Directory of pgPool status file
                                   # (change requires restart)
connection_cache = on
                                   # Activate connection pools
                                   # (change requires restart)
                                   # Semicolon separated list of queries
                                   # to be issued at the end of a session
                                   # The default is for 8.3 and later
reset_query_list = 'ABORT; DISCARD ALL'
                                   # The following one is for 8.2 and before
replication_mode = off
                                   # Activate replication mode
                                   # (change requires restart)
replicate_select = off
                                   # Replicate SELECT statements
                                   # when in replication mode
                                   # replicate_select is higher priority than
                                   # load_balance_mode.
insert_lock = on
                                   # Automatically locks a dummy row or a table
                                   # with INSERT statements to keep SERIAL data
                                   # consistency
                                   # Without SERIAL, no lock will be issued
lobj_lock_table = ''
                                   # When rewriting lo_creat command in
                                   # replication mode, specify table name to
                                   # lock
replication_stop_on_mismatch = off
                                   # On disagreement with the packet kind
                                   # sent from backend, degenerate the node
                                   # which is most likely "minority"
                                   # If off, just force to exit this session
failover_if_affected_tuples_mismatch = off
                                   # On disagreement with the number of affected
                                   # tuples in UPDATE/DELETE queries, then
                                   # degenerate the node which is most likely
                                   # "minority".
                                   # If off, just abort the transaction to
                                   # keep the consistency
load_balance_mode = on
                                   # Activate load balancing mode
                                   # (change requires restart)
ignore_leading_white_space = on
                                   # Ignore leading white spaces of each query
white_function_list = ''
                                   # Comma separated list of function names
                                   # that don't write to database
                                   # Regexp are accepted
black_function_list = 'currval,lastval,nextval,setval'
                                   # Comma separated list of function names
                                   # that write to database
                                   # Regexp are accepted
black_query_pattern_list = ''
                                   # Semicolon separated list of query patterns
                                   # that should be sent to primary node
                                   # Regexp are accepted
                                   # valid for streaming replicaton mode only.
database_redirect_preference_list = ''
                                   # comma separated list of pairs of database and node id.
                                   # example: postgres:primary,mydb[0-4]:1,mydb[5-9]:2'
                                   # valid for streaming replicaton mode only.
app_name_redirect_preference_list = ''
                                   # comma separated list of pairs of app name and node id.
                                   # example: 'psql:primary,myapp[0-4]:1,myapp[5-9]:standby'
                                   # valid for streaming replicaton mode only.
allow_sql_comments = off
                                   # if on, ignore SQL comments when judging if load balance or
                                   # query cache is possible.
                                   # If off, SQL comments effectively prevent the judgment
                                   # (pre 3.4 behavior).
disable_load_balance_on_write = 'transaction'
                                   # Load balance behavior when write query is issued
                                   # in an explicit transaction.
                                   # Note that any query not in an explicit transaction
                                   # is not affected by the parameter.
                                   # 'transaction' (the default): if a write query is issued,
                                   # subsequent read queries will not be load balanced
                                   # until the transaction ends.
                                   # 'trans_transaction': if a write query is issued,
                                   # subsequent read queries in an explicit transaction
                                   # will not be load balanced until the session ends.
                                   # 'always': if a write query is issued, read queries will
                                   # not be load balanced until the session ends.
statement_level_load_balance = off
                                   # Enables statement level load balancing
master_slave_mode = on
                                   # Activate master/slave mode
                                   # (change requires restart)
#postgres复制模式  流复制
master_slave_sub_mode = 'stream'

sr_check_period = 5
                                   # Streaming replication check period
                                   # Disabled (0) by default
#流复制检查用户
sr_check_user = 'repuser'

#流复制检查密码
sr_check_password = 'repuser'

#流复制检查数据库
sr_check_database = 'postgres'

delay_threshold = 0
                                   # Threshold before not dispatching query to standby node
                                   # Unit is in bytes
                                   # Disabled (0) by default
follow_master_command = ''
                                   # Executes this command after master failover
                                   # Special values:
                                   #   %d = failed node id
                                   #   %h = failed node host name
                                   #   %p = failed node port number
                                   #   %D = failed node database cluster path
                                   #   %m = new master node id
                                   #   %H = new master node hostname
                                   #   %M = old master node id
                                   #   %P = old primary node id
                                   #   %r = new master port number
                                   #   %R = new master database cluster path
                                   #   %N = old primary node hostname
                                   #   %S = old primary node port number
                                   #   %% = '%' character
health_check_period = 10
                                   # Health check period
                                   # Disabled (0) by default
health_check_timeout = 20
                                   # Health check timeout
                                   # 0 means no timeout
#后端数据库健康检查用户
health_check_user = 'postgres'

#后端数据库健康检查密码
health_check_password = 'postgres'

#后端数据库健康检查数据库名称
health_check_database = 'postgres'

health_check_max_retries = 0
                                   # Maximum number of times to retry a failed health check before giving up.
health_check_retry_delay = 1
                                   # Amount of time to wait (in seconds) between retries.
connect_timeout = 10000
                                   # Timeout value in milliseconds before giving up to connect to backend.
                                   # Default is 10000 ms (10 second). Flaky network user may want to increase
                                   # the value. 0 means no timeout.
                                   # Note that this value is not only used for health check,
                                   # but also for ordinary conection to backend.
#对后端数据库健康检查失败的时候 执行的脚本，也就是当master节点宕机了，此时会从slave节点中选出一个hostname作为new_master,然后执行command new_master
failover_command = '/etc/pgpool-II/failover_stream.sh %H'
                                   # Executes this command at failover
                                   # Special values:
                                   #   %d = failed node id
                                   #   %h = failed node host name
                                   #   %p = failed node port number
                                   #   %D = failed node database cluster path
                                   #   %m = new master node id
                                   #   %H = new master node hostname
                                   #   %M = old master node id
                                   #   %P = old primary node id
                                   #   %r = new master port number
                                   #   %R = new master database cluster path
                                   #   %N = old primary node hostname
                                   #   %S = old primary node port number
                                   #   %% = '%' character
failback_command = ''
                                   # Executes this command at failback.
                                   # Special values:
                                   #   %d = failed node id
                                   #   %h = failed node host name
                                   #   %p = failed node port number
                                   #   %D = failed node database cluster path
                                   #   %m = new master node id
                                   #   %H = new master node hostname
                                   #   %M = old master node id
                                   #   %P = old primary node id
                                   #   %r = new master port number
                                   #   %R = new master database cluster path
                                   #   %N = old primary node hostname
                                   #   %S = old primary node port number
                                   #   %% = '%' character
failover_on_backend_error = on
                                   # Initiates failover when reading/writing to the
                                   # backend communication socket fails
                                   # If set to off, pgpool will report an
                                   # error and disconnect the session.
detach_false_primary = off
                                   # Detach false primary if on. Only
                                   # valid in streaming replicaton
                                   # mode and with PostgreSQL 9.6 or
                                   # after.
search_primary_node_timeout = 300
                                   # Timeout in seconds to search for the
                                   # primary node when a failover occurs.
                                   # 0 means no timeout, keep searching
                                   # for a primary node forever.
auto_failback = off
                                   # Dettached backend node reattach automatically
                                   # if replication_state is 'streaming'.
auto_failback_interval = 60
                                   # Min interval of executing auto_failback in
                                   # seconds.
recovery_user = 'nobody'
                                   # Online recovery user
recovery_password = ''
                                   # Online recovery password
                                   # Leaving it empty will make Pgpool-II to first look for the
                                   # Password in pool_passwd file before using the empty password
recovery_1st_stage_command = ''
                                   # Executes a command in first stage
recovery_2nd_stage_command = ''
                                   # Executes a command in second stage
recovery_timeout = 90
                                   # Timeout in seconds to wait for the
                                   # recovering node's postmaster to start up
                                   # 0 means no wait
client_idle_limit_in_recovery = 0
                                   # Client is disconnected after being idle
                                   # for that many seconds in the second stage
                                   # of online recovery
                                   # 0 means no disconnection
                                   # -1 means immediate disconnection
use_watchdog = on
                                    # Activates watchdog
                                    # (change requires restart)
trusted_servers = ''
                                    # trusted server list which are used
                                    # to confirm network connection
                                    # (hostA,hostB,hostC,...)
                                    # (change requires restart)
ping_path = '/bin'
                                    # ping command path
                                    # (change requires restart)
#watchdog（个人理解就是一个监听器） 名称
wd_hostname = 'pmotest1'

#watchdog 端口
wd_port = 9000

#watchdog 优先级（当获取虚拟ip的pgpool节点宕机了之后，通过选举选出新的pgpool master）
wd_priority = 16
                                    # priority of this watchdog in leader election
                                    # (change requires restart)
wd_authkey = ''
                                    # Authentication key for watchdog communication
                                    # (change requires restart)
wd_ipc_socket_dir = '/tmp'
                                    # Unix domain socket path for watchdog IPC socket
                                    # The Debian package defaults to
                                    # /var/run/postgresql
                                    # (change requires restart)
#虚拟ip
delegate_IP = '192.168.64.20'
                                    # delegate IP address
                                    # If this is empty, virtual IP never bring up.
                                    # (change requires restart)
if_cmd_path = '/sbin'
                                    # path to the directory where if_up/down_cmd exists
                                    # If if_up/down_cmd starts with "/", if_cmd_path will be ignored.
                                    # (change requires restart)
#启动虚拟ip的命令
if_up_cmd = '/sbin/ip addr add $_IP_$/24 dev eth0 label eth0:0'

#关闭虚拟ip的命令
if_down_cmd = '/sbin/ip addr del $_IP_$/24 dev eth0'


arping_path = '/usr/sbin'
                                    # arping command path
                                    # If arping_cmd starts with "/", if_cmd_path will be ignored.
                                    # (change requires restart)
arping_cmd = '/usr/sbin/arping -U $_IP_$ -w 1 -I eth0'
                                    # arping command
                                    # (change requires restart)
clear_memqcache_on_escalation = on
                                    # Clear all the query cache on shared memory
                                    # when standby pgpool escalate to active pgpool
                                    # (= virtual IP holder).
                                    # This should be off if client connects to pgpool
                                    # not using virtual IP.
                                    # (change requires restart)
wd_escalation_command = ''
                                    # Executes this command at escalation on new active pgpool.
                                    # (change requires restart)
wd_de_escalation_command = ''
                                    # Executes this command when master pgpool resigns from being master.
                                    # (change requires restart)
failover_when_quorum_exists = on
                                    # Only perform backend node failover
                                    # when the watchdog cluster holds the quorum
                                    # (change requires restart)
failover_require_consensus = on
                                    # Perform failover when majority of Pgpool-II nodes
                                    # aggrees on the backend node status change
                                    # (change requires restart)
allow_multiple_failover_requests_from_node = off
                                    # A Pgpool-II node can cast multiple votes
                                    # for building the consensus on failover
                                    # (change requires restart)
enable_consensus_with_half_votes = off
                                    # apply majority rule for consensus and quorum computation
                                    # at 50% of votes in a cluster with even number of nodes.
                                    # when enabled the existence of quorum and consensus
                                    # on failover is resolved after receiving half of the
                                    # total votes in the cluster, otherwise both these
                                    # decisions require at least one more vote than
                                    # half of the total votes.
                                    # (change requires restart)
wd_monitoring_interfaces_list = ''  # Comma separated list of interfaces names to monitor.
                                    # if any interface from the list is active the watchdog will
                                    # consider the network is fine
                                    # 'any' to enable monitoring on all interfaces except loopback
                                    # '' to disable monitoring
                                    # (change requires restart)
wd_lifecheck_method = 'heartbeat'
                                    # Method of watchdog lifecheck ('heartbeat' or 'query' or 'external')
                                    # (change requires restart)
wd_interval = 5
                                    # lifecheck interval (sec) > 0
                                    # (change requires restart)
wd_heartbeat_port = 9694
                                    # Port number for receiving heartbeat signal
                                    # (change requires restart)
wd_heartbeat_keepalive = 2
                                    # Interval time of sending heartbeat signal (sec)
                                    # (change requires restart)
wd_heartbeat_deadtime = 30
                                    # Deadtime interval for heartbeat signal (sec)
                                    # (change requires restart)
#其他pgpool watchdog 中的第一个节点的hostname
heartbeat_destination0 = 'pmotest2'
#其他pgpool watchdog 中的第一个节点的端口
heartbeat_destination_port0 = 9694
#其他pgpool watchdog 中的第一个节点的网卡名称
heartbeat_device0 = 'eth0'

#同上
heartbeat_destination1 = 'pmotest3'
heartbeat_destination_port1 = 9694
heartbeat_device1 = 'eth0'

#同上
heartbeat_destination2 = 'pmotest4'
heartbeat_destination_port2 = 9694
heartbeat_device2 = 'eth0'

wd_life_point = 3
                                    # lifecheck retry times
                                    # (change requires restart)
wd_lifecheck_query = 'SELECT 1'
                                    # lifecheck query to pgpool from watchdog
                                    # (change requires restart)
wd_lifecheck_dbname = 'template1'
                                    # Database name connected for lifecheck
                                    # (change requires restart)
wd_lifecheck_user = 'nobody'
                                    # watchdog user monitoring pgpools in lifecheck
                                    # (change requires restart)
wd_lifecheck_password = ''
                                    # Password for watchdog user in lifecheck
                                    # Leaving it empty will make Pgpool-II to first look for the
                                    # Password in pool_passwd file before using the empty password
                                    # (change requires restart)
#其他的pgpool hostname
other_pgpool_hostname0 = 'pmotest2'
other_pgpool_port0 = 9999
other_wd_port0 = 9000

#同上
other_pgpool_hostname1 = 'pmotest3'
other_pgpool_port1 = 9999
other_wd_port1 = 9000

#同上
other_pgpool_hostname2 = 'pmotest4'
other_pgpool_port2 = 9999
other_wd_port2 = 9000

relcache_expire = 0
                                   # Life time of relation cache in seconds.
                                   # 0 means no cache expiration(the default).
                                   # The relation cache is used for cache the
                                   # query result against PostgreSQL system
                                   # catalog to obtain various information
                                   # including table structures or if it's a
                                   # temporary table or not. The cache is
                                   # maintained in a pgpool child local memory
                                   # and being kept as long as it survives.
                                   # If someone modify the table by using
                                   # ALTER TABLE or some such, the relcache is
                                   # not consistent anymore.
                                   # For this purpose, cache_expiration
                                   # controls the life time of the cache.
relcache_size = 256
                                   # Number of relation cache
                                   # entry. If you see frequently:
                                   # "pool_search_relcache: cache replacement happend"
                                   # in the pgpool log, you might want to increate this number.
check_temp_table = catalog
                                   # Temporary table check method. catalog, trace or none.
                                   # Default is catalog.
check_unlogged_table = on
                                   # If on, enable unlogged table check in SELECT statements.
                                   # This initiates queries against system catalog of primary/master
                                   # thus increases load of master.
                                   # If you are absolutely sure that your system never uses unlogged tables
                                   # and you want to save access to primary/master, you could turn this off.
                                   # Default is on.
enable_shared_relcache = on
                                   # If on, relation cache stored in memory cache,
                                   # the cache is shared among child process.
                                   # Default is on.
                                   # (change requires restart)
relcache_query_target = master     # Target node to send relcache queries. Default is master (primary) node.
                                   # If load_balance_node is specified, queries will be sent to load balance node.
memory_cache_enabled = off
                                   # If on, use the memory cache functionality, off by default
                                   # (change requires restart)
memqcache_method = 'shmem'
                                   # Cache storage method. either 'shmem'(shared memory) or
                                   # 'memcached'. 'shmem' by default
                                   # (change requires restart)
memqcache_memcached_host = 'localhost'
                                   # Memcached host name or IP address. Mandatory if
                                   # memqcache_method = 'memcached'.
                                   # Defaults to localhost.
                                   # (change requires restart)
memqcache_memcached_port = 11211
                                   # Memcached port number. Mondatory if memqcache_method = 'memcached'.
                                   # Defaults to 11211.
                                   # (change requires restart)
memqcache_total_size = 67108864
                                   # Total memory size in bytes for storing memory cache.
                                   # Mandatory if memqcache_method = 'shmem'.
                                   # Defaults to 64MB.
                                   # (change requires restart)
memqcache_max_num_cache = 1000000
                                   # Total number of cache entries. Mandatory
                                   # if memqcache_method = 'shmem'.
                                   # Each cache entry consumes 48 bytes on shared memory.
                                   # Defaults to 1,000,000(45.8MB).
                                   # (change requires restart)
memqcache_expire = 0
                                   # Memory cache entry life time specified in seconds.
                                   # 0 means infinite life time. 0 by default.
                                   # (change requires restart)
memqcache_auto_cache_invalidation = on
                                   # If on, invalidation of query cache is triggered by corresponding
                                   # DDL/DML/DCL(and memqcache_expire).  If off, it is only triggered
                                   # by memqcache_expire.  on by default.
                                   # (change requires restart)
memqcache_maxcache = 409600
                                   # Maximum SELECT result size in bytes.
                                   # Must be smaller than memqcache_cache_block_size. Defaults to 400KB.
                                   # (change requires restart)
memqcache_cache_block_size = 1048576
                                   # Cache block size in bytes. Mandatory if memqcache_method = 'shmem'.
                                   # Defaults to 1MB.
                                   # (change requires restart)
memqcache_oiddir = '/var/log/pgpool/oiddir'
                                   # Temporary work directory to record table oids
                                   # (change requires restart)
white_memqcache_table_list = ''
                                   # Comma separated list of table names to memcache
                                   # that don't write to database
                                   # Regexp are accepted
black_memqcache_table_list = ''
                                   # Comma separated list of table names not to memcache
                                   # that don't write to database
                                   # Regexp are accepted
```
### 配置机器之间的ssh认证
...省略
### 附件
[配置文件下载](https://github.com/luxiang0412/blog/raw/master/source/_posts/postgres-%E6%B5%81%E5%A4%8D%E5%88%B6-pgpool-II-%E4%B8%BB%E4%BB%8E%E7%83%AD%E5%A4%87/%E7%9B%B8%E5%85%B3%E7%9A%84%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6.tar)
### 提示

    提示
    先启动pg数据库，然后再启动pgpool
    先停止pgpool，再停止pg数据库


### 参考
- [postgres install](https://www.postgresql.org/download/linux/redhat/)
- [Streaming_Replication](https://wiki.postgresql.org/wiki/Streaming_Replication)
- [pgpool Wiki](https://www.pgpool.net/mediawiki/index.php/Main_Page)
- [pgpool-II 4.1.1 document](https://www.pgpool.net/docs/latest/en/html/index.html)
- [Pgpool-II + Watchdog Setup Example](https://www.pgpool.net/docs/latest/en/html/example-cluster.html)
- [PGPool-II+PG流复制实现HA主备切换](https://www.jianshu.com/p/ef183d0a9213)