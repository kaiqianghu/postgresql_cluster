# 架构图

![1673249476708.png](https://cdn.nlark.com/yuque/0/2023/png/5365995/1673249478996-c897cc90-b08f-4735-bc29-9a2caa60f624.png#averageHue=%23faf9f7&clientId=u57a32249-7b3e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=222&id=TnMpO&margin=%5Bobject%20Object%5D&name=1673249476708.png&originHeight=222&originWidth=613&originalType=binary&ratio=1&rotation=0&showTitle=false&size=12703&status=done&style=none&taskId=u108d72f1-b151-49f5-9ad9-638670e742f&title=&width=613)

# 一、PostgreSQL9.6.9 源码安装

**注：在slave1、slave2分别安装postgre服务，端口与主库端口一致18889**

## 1、安装数据库

### 1）安装编译依赖库

```shell
yum -y install gcc gcc-c++ make readline-devel zlib-devel
```

### 2）创建管理用户和组

```shell
# 创建用户组和用户
groupadd pgAdmin && useradd pgAdmin -d /home/pgAdmin -g pgAdmin

# 普通账户提权
echo "pgAdmin ALL=(ALL)       NOPASSWD:ALL" >> /etc/sudoers
```

### 3）编译安装pg

```shell
mkdir /usr/local/postgresql_slave/

tar -xvf postgresql-9.6.9.tar.gz -C /usr/local/postgresql_slave/

cd /usr/local/postgresql_slave/postgresql-9.6.9

# 创建数据库pg_data目录
mkdir /usr/local/postgresql_slave/pg_data

# 编译
./configure

# 安装
make -j4 && make install
```

## 2、修改数据库配置

### 1）修改数据库pg_data目录权限

```shell
chown -R pgAdmin:pgAdmin /usr/local/postgresql_slave/
```

### 2）初始化数据库

```shell
su - pgAdmin

# 初始化数据库
/usr/pgsql-9.6/bin/initdb -D /usr/local/postgresql_slave/pg_data
```

## 3、修改配置

### 1）配置postgresql.conf

```shell
# 备份配置文件
cp /usr/local/postgresql_slave/pg_data/postgresql.conf /usr/local/postgresql_slave/pg_data/postgresql.conf.bak 
```

vim /usr/local/postgresql_slave/pg_data/postgresql.conf

```shell
listen_addresses = '*'          # what IP address(es) to listen on;
port = 18889                            # (change requires restart)
max_connections = 3000                  # (change requires restart)
shared_buffers = 128GB                  # min 128kB
log_destination = 'stderr'              # Valid values are combinations of
logging_collector = on                  # Enable capturing of stderr and csvlog
log_directory = 'pg_log'                # directory where log files are written,
log_filename = 'postgresql-%a.log'      # log file name pattern,
log_truncate_on_rotation = on           # If on, an existing log file with the
log_rotation_age = 1d                   # Automatic rotation of logfiles will
log_rotation_size = 0                   # Automatic rotation of logfiles will
log_line_prefix = '< %m > '                     # special values:
log_timezone = 'PRC'
search_path = 'back_end,fore_end,public'#default_tablespace = ''                # a tablespace name, '' uses the default
datestyle = 'iso, mdy'
timezone = 'PRC'
lc_messages = 'en_US.UTF-8'                     # locale for system error message
lc_monetary = 'en_US.UTF-8'                     # locale for monetary formatting
lc_numeric = 'en_US.UTF-8'                      # locale for number formatting
lc_time = 'en_US.UTF-8'                         # locale for time formatting
default_text_search_config = 'pg_catalog.english'
```

cat /usr/local/postgresql_slave/pg_data/postgresql.conf

### 2）配置pg_hba.conf

vim /usr/local/postgresql_slave/pg_data/pg_hba.conf

```shell
echo "local   postgres   all                               trust
host   all         all   0.0.0.0   0.0.0.0   md5" >> /usr/local/postgresql_slave/pg_data/pg_hba.conf

cat /usr/local/postgresql_slave/pg_data/pg_hba.conf
```

### 3）配置环境变量，追加

```shell
echo "PATH=$PATH:$HOME/.local/bin:$HOME/bin
PATH=$PATH:/usr/pgsql-9.6/bin
export PGDATA=/home/pg/postgresql_slave/pg_data" >> /home/pgAdmin/.bash_profile
```

```shell
# 修改权限
chown pgAdmin.pgAdmin /home/pgAdmin/.bash_profile

chmod 644 /home/pgAdmin/.bash_profile
```

### 4）配置system管理postgre服务

vim  /usr/lib/systemd/system/postgresql-cluster.service

```shell
cat > /usr/lib/systemd/system/postgresql-cluster.service << EOF
[Unit]
Description=PostgreSQL 9.6 database server
Documentation=https://www.postgresql.org/docs/9.6/static/
After=syslog.target
After=network.target

[Service]
Type=notify
User=pgAdmin
Group=pgAdmin
Environment=PGDATA=/usr/local/postgresql_slave/pg_data/
ExecStartPre=/usr/pgsql-9.6/bin/postgresql96-check-db-dir \${PGDATA}
ExecStart=/usr/pgsql-9.6/bin/postmaster -D \${PGDATA}
ExecReload=/bin/kill -HUP \$MAINPID
KillMode=mixed
KillSignal=SIGINT
TimeoutSec=0

[Install]
WantedBy=multi-user.target
EOF
```

cat /usr/lib/systemd/system/postgresql-cluster.service

```shell
chmod 640 /usr/lib/systemd/system/postgresql-cluster.service

chown pgAdmin.pgAdmin /usr/lib/systemd/system/postgresql-cluster.service
```

## 4、pg数据库服务管理

```shell
chown -R pgAdmin.pgAdmin /usr/local/postgresql_slave/

chmod 777 /var/run/postgresql/
```

```shell
systemctl daemon-reload
# 查看服务状态
sudo systemctl status postgresql-cluster.service

# 设置服务开机自启
sudo systemctl enable postgresql-cluster.service

# 启动服务
sudo systemctl start postgresql-cluster.service

# 重启服务
sudo systemctl restart postgresql-cluster.service

# 停止服务
sudo systemctl stop postgresql-cluster.service
```

## 5、服务启动故障处理方案

### 5.1 FATAL:  could not create semaphores: No space left on device

```shell
# cat  /proc/sys/kernel/sem
SEMMSL   SEMMNS         SEMOPM  SEMMNI
50100   128256000       50100   2560

SEMMSL 每个信号数量set中信号数量最大个数
SEMMNS linux系统中信号最大个数
SEMOPM semop系统调使用许的信号最大数设置，设置和SEMMSL一样即可
SEMMNI linux系统信息量集最大个数
```

```shell
# 修改 vi /etc/sysctl.conf 的以下参数
echo "kernel.sem =5010 641280 5010 626" >> /etc/sysctl.conf 

# 生效
sysctl -p
```

### 5.２FATAL:  could not map anonymous shared memory: Cannot allocate memory

内存不够用于分配shared memory大小，即shared memory设置超出了所能分配到的内存大小　👇

```shell
# 根据性能优化策略,建议shared_buffers参数值设置为本机物理内在的1/4。
# 把shared_buffers重新设置成4G，保存并退出。重启pg，一切正常。

# vi /usr/local/postgresql_slave/pg_data/postgresql.conf
shared_buffers = 64GB 
```

# 