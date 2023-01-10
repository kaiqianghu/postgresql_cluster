# 二、配置数据库主从

**注：slave1为异步复制，slave2为同步复制**

## 1、master操作

### 1）创建归档目录

```shell
mkdir -p /usr/local/postgresql_slave/pg_data/pg_archive
```

### 2）创建replica用户进行主从同步

```shell
# 修改现网的pg_hba.conf，以便创建role
host   all         all   0.0.0.0   0.0.0.0   md5
改为
host   all         all   0.0.0.0   0.0.0.0   trust

# 重启数据库服务
systemctl restart postgresql-cluster.service

# 连接数据库创建角色
psql -h master地址 -p master端口 -U pgAdmin -d postgres

postgres=# create role 角色名 login replication encrypted password '密码';
```

### 3）修改pg_hba.conf，允许replica用户来同步

vim /usr/local/postgresql_slave/pg_data/pg_hba.conf

```shell
# 允许从服务器使用replica用户来复制
host    replication     replica         master/32        md5
host    replication     replica         slave1/32        md5
host    replication     replica         slave2/32        md5
```

### 4）修改postgresql.conf

vim /usr/local/postgresql_slave/pg_data/postgresql.conf

```shell
listen_addresses = '*'
port = 18889
max_connections = 3000
unix_socket_permissions = 0770
shared_buffers = 128GB
dynamic_shared_memory_type = sysv
wal_level = hot_standby
archive_mode = on
archive_command = 'cp %p /usr/local/postgresql_slave/pg_data/pg_archive/%f'
max_wal_senders = 10
wal_keep_segments = 256
wal_sender_timeout = 60s
synchronous_standby_names = 'slave1'
synchronous_commit = off
hot_standby = on
log_destination = 'stderr'
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%a.log'
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
log_line_prefix = '< %m > '
log_timezone = 'PRC'
search_path = 'back_end,fore_end,public'
datestyle = 'iso, mdy'
timezone = 'PRC'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'
```

### 5）配置.pgpass密码文件

```shell
# 主节点(配置.pgpass密码文件)
vim /home/pg/.pgpass
master地址:端口:数据库:角色名:密码
slave1地址:端口:数据库:角色名:密码
slave2地址:端口:数据库:角色名:密码


vim /root/.pgpass
master地址:端口:数据库:角色名:密码
slave1地址:端口:数据库:角色名:密码
slave2地址:端口:数据库:角色名:密码

chmod 0600 /root/.pgpass
chmod 0600 /home/pgAdmin/.pgpass
```

### 6）重启postgres

```shell
# 重启数据库
sudo systemctl restart postgresql-cluster.service
```

## 3、主从同步复制slave1操作

**注意：slave1同步master数据**

### 1）将主库数据拷贝至从库

```shell
systemctl stop postgresql-cluster.service

su - pgAdmin

cd /usr/local/postgresql_slave/

rm -rf pg_data

#  基础备份(从主库备份,-h指定数据库地址，-p指定端口，-U指定用户)
/usr/pgsql-9.6/bin/pg_basebackup -h master地址 -p master端口 -U 同步角色名 -v -R -D /usr/local/postgresql_slave/pg_data -X stream -P

mkdir -p /usr/local/postgresql_slave/pg_data/pg_archive
```

### 2）配置recovery.conf 👇

```shell
vim /usr/local/postgresql_slave/pg_data/recovery.conf

# 说明该节点是从服务器
standby_mode = on    
# 主节点的信息以及连接的用户
primary_conninfo = 'application_name=slave1 host=master地址 port=master端口 user=同步角色名 password=同步角色密码 sslmode=prefer sslcompression=1'
# 同步到最新数据
recovery_target_timeline = 'latest'
```

```shell
standby_mode = on
primary_conninfo = 'application_name=slave1 host=master地址 port=master端口 user=同步角色名 password=同步角色密码 sslmode=prefer sslcompression=1'
recovery_target_timeline = 'latest'

```

### 3）修改pg_hba.conf，允许replica用户来同步

vim /usr/local/postgresql_slave/pg_data/pg_hba.conf

```shell
# 允许从服务器使用replica用户来复制
host    replication     replica         master/32        md5
host    replication     replica         slave1/32        md5
host    replication     replica         slave2/32        md5
```

cat /usr/local/postgresql_slave/pg_data/pg_hba.conf

### 4）配置postgresql.conf

vim /usr/local/postgresql_slave/pg_data/postgresql.conf

```shell
listen_addresses = '*'
port = 18889
max_connections = 3000
unix_socket_permissions = 0770
shared_buffers = 128GB
dynamic_shared_memory_type = sysv
wal_level = hot_standby
archive_mode = on
archive_command = 'cp %p /usr/local/postgresql_slave/pg_data/pg_archive/%f'
max_wal_senders = 10
wal_keep_segments = 256
wal_sender_timeout = 60s
synchronous_standby_names = 'slave2'
synchronous_commit = off
hot_standby = on
log_destination = 'stderr'
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%a.log'
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
log_line_prefix = '< %m > '
log_timezone = 'PRC'
search_path = 'back_end,fore_end,public'
datestyle = 'iso, mdy'
timezone = 'PRC'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'
```

### 5）配置.pgpass密码文件

```shell
# 备节点(配置.pgpass密码文件)
vim /home/pgAdmin/.pgpass
master地址:端口:数据库:角色名:密码
slave1地址:端口:数据库:角色名:密码
slave2地址:端口:数据库:角色名:密码


vim /root/.pgpass
master地址:端口:数据库:角色名:密码
slave1地址:端口:数据库:角色名:密码
slave2地址:端口:数据库:角色名:密码

chmod 0600 /root/.pgpass
chmod 0600 /home/pgAdmin/.pgpass
```

### 6）重启postgres

```shell
# 基于root账号做的基础备份，需要将相关目录文件的权限变更
chown -R pgAdmin:pgAdmin /usr/local/postgresql_slave/

# 重启数据库
sudo systemctl restart postgresql-cluster.service
```

### 7）使用验证

#### 7.1 查看程序进程

```shell
# 1、主库sender进程
ps -ef | grep sender

pg        17124  11068  3 22:55 ?        00:00:05 postgres: wal sender process replica 10.125.26.2(44806) streaming 49/7C578070

# 2、备库receiver进程
ps -ef | grep receiver

pgAdmin   17462  17448  2 22:55 ?        00:00:05 postgres: wal receiver process   streaming 49/7E088168
```

#### 7.2 查看复制状态

```shell
####################### 连接数据库（主库）#######################
psql -h master地址 -p master端口 -U 角色名 -d postgres

postgres=# \x
postgres=# select * from pg_stat_replication;

####################### 连接数据库（备库）#######################
psql -h master地址 -p master端口 -U 角色名 -d postgres

# 备库WAL接收进程输出
postgres=# \x

postgres=# select * from pg_stat_get_wal_receiver();
```

## 4、主从同步复制slave2操作

**注意：slave2同步slave1数据**

### 1）将slave1数据拷贝至slave2

```shell
systemctl stop postgresql-cluster.service

su - pgAdmin

cd /usr/local/postgresql_slave/

rm -rf pg_data

#  基础备份
/usr/pgsql-9.6/bin/pg_basebackup -h Slave1地址 -p Slave1端口 -U replica -v -R -D /usr/local/postgresql_slave/pg_data -X stream -P

mkdir -p /usr/local/postgresql_slave/pg_data/pg_archive
```

### 2）配置recovery.conf 👇

vim /usr/local/postgresql_slave/pg_data/recovery.conf

```shell
# 说明该节点是从服务器
standby_mode = on    
# 主节点的信息以及连接的用户
primary_conninfo = 'application_name=slave2 host=Slave1地址 port=Slave1端口 user=Slave1角色名 password=Slave1密码 sslmode=prefer sslcompression=1'
# 同步到最新数据
recovery_target_timeline = 'latest'
```

```shell
standby_mode = on    
primary_conninfo = 'application_name=slave2 host=Slave1地址 port=Slave1端口 user=Slave1角色名 password=Slave1密码 sslmode=prefer sslcompression=1'
recovery_target_timeline = 'latest'
```

### 3）修改pg_hba.conf，允许replica用户来同步

vim /usr/local/postgresql_slave/pg_data/pg_hba.conf

```shell
# 允许从服务器使用replica用户来复制
host    replication     replica         master/32        md5
host    replication     replica         slave1/32        md5
host    replication     replica         slave2/32        md5
```

### 4）配置postgresql.conf

vim /usr/local/postgresql_slave/pg_data/postgresql.conf

```shell
listen_addresses = '*'
port = 18889
max_connections = 3000
unix_socket_permissions = 0770
shared_buffers = 128GB
dynamic_shared_memory_type = sysv
wal_level = hot_standby
archive_mode = on
archive_command = 'cp %p /usr/local/postgresql_slave/pg_data/pg_archive/%f'
max_wal_senders = 10
wal_keep_segments = 256
wal_sender_timeout = 60s
synchronous_standby_names = 'slave3'
synchronous_commit = off
hot_standby = on
log_destination = 'stderr'
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%a.log'
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
log_line_prefix = '< %m > '
log_timezone = 'PRC'
search_path = 'back_end,fore_end,public'
datestyle = 'iso, mdy'
timezone = 'PRC'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'
```

### 5）配置.pgpass密码文件

```shell
# 备节点(配置.pgpass密码文件)
vim /home/pgAdmin/.pgpass
master地址:端口:数据库:角色名:密码
slave1地址:端口:数据库:角色名:密码
slave2地址:端口:数据库:角色名:密码


vim /root/.pgpass
master地址:端口:数据库:角色名:密码
slave1地址:端口:数据库:角色名:密码
slave2地址:端口:数据库:角色名:密码

chmod 0600 /root/.pgpass
chmod 0600 /home/pgAdmin/.pgpass
```

### 6）重启postgres

```shell
# 基于root账号做的基础备份，需要将相关目录文件的权限变更
chown -R pgAdmin:pgAdmin /usr/local/postgresql_slave/

# 重启数据库
sudo systemctl restart postgresql-cluster.service
```

### 7）使用验证

#### 7.1 查看程序进程

```shell
# 1、主库sender进程
ps -ef | grep sender

pg        17124  11068  3 22:55 ?        00:00:05 postgres: wal sender process replica 10.125.26.2(44806) streaming 49/7C578070

# 2、备库receiver进程
ps -ef | grep receiver

pgAdmin   17462  17448  2 22:55 ?        00:00:05 postgres: wal receiver process   streaming 49/7E088168
```

#### 7.2 查看复制状态

```shell
####################### 连接数据库（主库）#######################
psql -h Slave1地址 -p 18889 -U replica -d postgres

postgres=# \x
postgres=# select * from pg_stat_replication;

####################### 连接数据库（备库）#######################
psql -h Slave2地址 -p 18889 -U replica -d postgres

# 备库WAL接收进程输出
postgres=# \x

postgres=# select * from pg_stat_get_wal_receiver();
```

## 5 参数说明

### 1）pg_stat_replication参数说明

| 参数项           | 参数值                                                       |
| ---------------- | ------------------------------------------------------------ |
| pid              | WAL发送进程的进程号                                          |
| usename          | WAL发送进程的数据库用户名                                    |
| application_name | 连接WAL发送进程的应用别名，此参数显示值为备库recovery.conf配置文件中primary_conninfo参数application_name选项的值。 |
| client_addr      | 连接到WAL发送进程的客户端IP地址，也就是备库的IP              |
| backend_start    | WAL发送进程的启动时间                                        |
| state            | 显示WAL发送进程的状态：startup 表示WAL进程在启动过程中；catchup 表示备库正在追赶主库；streaming表示备库已经追赶上了主库，并且主库向备库发送WAL日志流，这个状态是流复制的常规状态；backup表示通过pg_basebackup正在进行备份；stopping表示WAL发送进程正在关闭； |
| sent_lsn         | WAL发送进程最近发送的WAL日志位置                             |
| write_lsn        | 备库最近写入的WAL日志位置，这时WAL日志流还在操作系统缓存中，还没写入备库WAL日志文件 |
| flush_lsn        | 备库最近写入的WAL日志位置，这时WAL日志流已写入备库WAL日志文件 |
| replay_lsn       | 备库最近应用的WAL日志位置                                    |
| write_lag        | 主库上WAL日志落盘后等待备库接收WAL日志（这是WAL日志流还没写入备库WAL日志文件，还在操作系统缓存中）并返回确认信息的时间 |
| flush_lag        | 主库上WAL日志落盘后等待备库接收WAL日志（这是WAL日志流还已写入备库WAL日志文件，但还没有应用WAL日志）并返回确认信息的时间 |
| replay_lag       | 主库上WAL日志落盘后等待备库接收WAL日志（这是WAL日志流还已写入备库WAL日志文件，并且已应用WAL日志）并返回确认信息的时间 |
| sync_priority    | 基于优先级的模式中备库被选中成为同步备库的优先级，对于基于quorum的选举模式此字段则无影响 |
| sync_state       | 同步状态 : async 表示备库为异步同步模式; potential 表示备库当前为异步同步模式，如果当前的同步备库宕机，异步备库可升级成为同步备库; sync 表示当前备库为同步模式; quorum 表示备库为quorum standbys 的候选; |

### 2）pg_stat_get_wal_receiver参数说明

| 参数项                | 参数值                                                       |
| --------------------- | ------------------------------------------------------------ |
| pid                   | WAL接收进程的进程号                                          |
| status                | WAL接收进程的状态                                            |
| receive_start_lsn     | WAL接收进程启动后使用的第一个WAL日志位置                     |
| receive_lsn           | 最近接收并写入WAL日志文件的WAL位置                           |
| last_msg_send_time    | 备库接收到发送进程最后一个消息后，向主库发回确认消息的发送时间 |
| last_msg_receipt_time | 备库接收到发送进程最后一个消息的接收时间                     |
| coninfo               | WAL接收进程使用的连接串，连接信息由备库$PGDATA目录的recovery.conf配置文件的primary_coninfo参数配置 |

