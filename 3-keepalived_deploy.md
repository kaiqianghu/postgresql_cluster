# 三、配置keepalived

## 1、keepalived(3台同样的操作)

### 1）keepalived

#### 1.1 上传源码包至要部署程序的服务器，进行程序部署

```shell

```

#### 1.2 配置依赖环境与源码安装keepalived，安装版本2.2.7

```shell
# 切换至root
sudo su -

# 安装依赖包
yum -y install gcc openssl-devel libnl libnl-devel libnfnetlink-devel

# 解压源码包
tar -xf keepalived-2.2.7.tar.gz -C /home/zzitcj/

cd /home/zzitcj/keepalived-2.2.7

# 配置Makefile，指向安装目录/usr/local/keepalived
./configure --prefix=/usr/local/keepalived --with-init=systemd --with-systemdsystemunitdir=/usr/lib/systemd/system

# 编译并安装
make && make install

chown -R pgAdmin.pgAdmin /usr/local/keepalived
```

#### 1.3 修改启动参数

```shell
sed -i 's@KEEPALIVED_OPTIONS=.*@KEEPALIVED_OPTIONS="-f /usr/local/keepalived/etc/keepalived/keepalived.conf -p /usr/local/keepalived/temp/keepalived.pid -D"@' /usr/local/keepalived/etc/sysconfig/keepalived
```

## 2、修改配置文件keepalived.conf

### 1）master

#### 1.1 编写keepalived.conf

##### master

vim /usr/local/keepalived/etc/keepalived/keepalived.conf

```shell
! Configuration File for keepalived
global_defs {
  router_id HOSTNAME
  vrrp_iptables
  vrrp_skip_check_adv_addr
  vrrp_strict
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}
vrrp_script check_pg_alived {
  script "/usr/local/keepalived/bin/pg_moniter.sh"
  interval 20
  fall 3    # require 3 failures for KO
}
vrrp_instance VI_1 {
  state MASTER
  interface bond1
  virtual_router_id 51
  priority 100
  advert_int 1
  nopreempt
  authentication {
    auth_type PASS
    auth_pass PG9CLUSTER
  }
  track_script {
    check_pg_alived
  }
  virtual_ipaddress {
    vipaddress
  }
  	notify_master /usr/local/keepalived/bin/active_standby.sh
}
```

##### slave1

vim /usr/local/keepalived/etc/keepalived/keepalived.conf

```shell
! Configuration File for keepalived
global_defs {
  router_id HOSTNAME
  vrrp_iptables			
  vrrp_skip_check_adv_addr		
  vrrp_strict			
  vrrp_garp_interval 0		
  vrrp_gna_interval 0	
}
vrrp_script check_pg_alived {
  script "/usr/local/keepalived/bin/pg_moniter.sh" 
  interval 20 
  fall 3    # require 3 failures for KO
} 
vrrp_instance VI_1 {
  state BACKUP
  interface bond1
  virtual_router_id 51
  priority 90
  advert_int 1
  nopreempt
  authentication {
    auth_type PASS
    auth_pass PG9CLUSTER
  }
  track_script { 
    check_pg_alived
  }
  virtual_ipaddress {   
    vipaddress
  }
  	notify_master /usr/local/keepalived/bin/active_standby.sh
}
```

##### slave2

vim /usr/local/keepalived/etc/keepalived/keepalived.conf

```shell
! Configuration File for keepalived
global_defs {
  router_id HOSTNAME
  vrrp_iptables			
  vrrp_skip_check_adv_addr		
  vrrp_strict			
  vrrp_garp_interval 0		
  vrrp_gna_interval 0	
}
vrrp_script check_pg_alived {
  script "/usr/local/keepalived/bin/pg_moniter.sh" 
  interval 20 
  fall 3    # require 3 failures for KO
} 
vrrp_instance VI_1 {
  state BACKUP
  interface bond1
  virtual_router_id 51
  priority 80
  advert_int 1
  nopreempt
  authentication {
    auth_type PASS
    auth_pass PG9CLUSTER
  }
  track_script { 
    check_pg_alived
  }
  virtual_ipaddress {   
    vipaddress
  }
  	notify_master /usr/local/keepalived/bin/active_standby.sh
}
```

#### 1.2 编写数据库监控脚本pg_monitor.sh

##### master

vim /usr/local/keepalived/bin/pg_moniter.sh

```shell
#!/bin/bash
export PGHOST=master库地址
export PGPORT=18889
export PGUSER=replica
export PGDBNAME=icpdb
export LANG=en_US.utf8
export PGDATA=/home/pg/pg_data/

MONITOR_LOG="/tmp/pg_monitor.log"
SQL='select 1;'

# 如果是备库,则退出，此脚本不检查备库存活状态
standby_flg=`psql -h $PGHOST -p $PGPORT -U $PGUSER -d postgres -At -c "select pg_is_in_recovery();"`

if [ ${standby_flg} == 't' ]; then
    echo -e "`date +%F\ %T`: 这是一个备库,此脚本不检查备库存活状态,退出!\n" >> $MONITOR_LOG
    exit 0
fi

# 判断自己端口是否可用
CMD=`ss -ntulp | grep 18889 | wc -l`
if [ $CMD -ge 2 ];then
    ret="yes"
else
    ret="no"
fi

# 判断能否连接数据库执行select查询
echo $SQL | psql -At -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDBNAME

if [[ $? -eq 0 ]] && [[ $ret = "yes" ]]; then
   echo -e "`date +%F\ %T`:  主库为健康状态."  >> $MONITOR_LOG
	 exit 0
else
   echo -e "`date +%F\ %T`:  主库状态异常,连续3次异常后keepalived会发生切换"  >> $MONITOR_LOG
   exit 1
fi
```

##### slave1

vim /usr/local/keepalived/bin/pg_moniter.sh

```shell
#!/bin/bash
export PGHOST=slave1库地址
export PGPORT=18889
export PGUSER=replica
export PGDBNAME=icpdb
export LANG=en_US.utf8
export PGDATA=/home/pg/pg_data/

MONITOR_LOG="/tmp/pg_monitor.log"
SQL='select 1;'

# 如果是备库,则退出，此脚本不检查备库存活状态
standby_flg=`psql -h $PGHOST -p $PGPORT -U $PGUSER -d postgres -At -c "select pg_is_in_recovery();"`
if [ ${standby_flg} == 't' ]; then
    echo -e "`date +%F\ %T`: 这是一个备库,此脚本不检查备库存活状态,退出!\n" >> $MONITOR_LOG
    exit 0
fi

# 判断自己端口是否可用
CMD=`ss -ntulp | grep 18889 | wc -l`
if [ $CMD -ge 2 ];then
    ret="yes"
else
    ret="no"
fi

# 判断能否连接数据库执行select查询
echo $SQL | psql -At -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDBNAME

if [[ $? -eq 0 ]] && [[ $ret = "yes" ]]; then
   echo -e "`date +%F\ %T`:  主库为健康状态."  >> $MONITOR_LOG
	 exit 0
else
   echo -e "`date +%F\ %T`:  主库状态异常,连续3次异常后keepalived会发生切换"  >> $MONITOR_LOG
   exit 1
fi
```

##### slave2

vim /usr/local/keepalived/bin/pg_moniter.sh

```shell
#!/bin/bash
export PGHOST=slave2库地址
export PGPORT=18889
export PGUSER=replica
export PGDBNAME=icpdb
export LANG=en_US.utf8
export PGDATA=/home/pg/pg_data/

MONITOR_LOG="/tmp/pg_monitor.log"
SQL='select 1;'

# 如果是备库,则退出，此脚本不检查备库存活状态
standby_flg=`psql -h $PGHOST -p $PGPORT -U $PGUSER -d postgres -At -c "select pg_is_in_recovery();"`
if [ ${standby_flg} == 't' ]; then
    echo -e "`date +%F\ %T`: 这是一个备库,此脚本不检查备库存活状态,退出!\n" >> $MONITOR_LOG
    exit 0
fi

# 判断自己端口是否可用
CMD=`ss -ntulp | grep 18889 | wc -l`
if [ $CMD -ge 2 ];then
    ret="yes"
else
    ret="no"
fi

# 判断能否连接数据库执行select查询
echo $SQL | psql -At -h $PGHOST -p $PGPORT -U $PGUSER -d $PGDBNAME

if [[ $? -eq 0 ]] && [[ $ret = "yes" ]]; then
   echo -e "`date +%F\ %T`:  主库为健康状态."  >> $MONITOR_LOG
	 exit 0
else
   echo -e "`date +%F\ %T`:  主库状态异常,连续3次异常后keepalived会发生切换"  >> $MONITOR_LOG
   exit 1
fi
```

#### 1.3 编写数据库切换脚本

##### master

vim /usr/local/keepalived/bin/active_standby.sh

```shell
#!/bin/bash
# 环境变量
source /home/postgres/.bash_profile
export PG_OS_USER=pg
MONITOR_LOG="/tmp/pg_monitor.log"

# pg_failover 函数，用于主库故障时激活从库
pg_failover()
{
# PROMOTE_STATUS 表示激活备库成功标志，1 表示失败，0 表示成功
PROMOTE_STATUS=1

# 激活备库
su - $PG_OS_USER -c "pg_ctl promote -D /usr/local/postgresql_slave/pg_data/"
if [ $? -eq 0 ]; then
  echo -e "`date +%F\ %T`: `hostname` promote promote 执行成功. " >> $MONITOR_LOG
  PROMOTE_STATUS=0
fi
if [ $PROMOTE_STATUS -ne 0 ]; then
  echo -e "`date +%F\ %T`: promote promote 执行失败." >> $MONITOR_LOG
  return $PROMOTE_STATUS
fi
  echo -e "`date +%F\ %T`: pg_failover() function 调用成功." >> $MONITOR_LOG
  return 0
}
pg_failover
```

##### slave1/slave2

vim /usr/local/keepalived/bin/active_standby.sh

```shell
#!/bin/bash
# 环境变量
export PG_OS_USER=pgAdmin
MONITOR_LOG="/tmp/pg_monitor.log"

# pg_failover 函数，用于主库故障时激活从库
pg_failover()
{
# PROMOTE_STATUS 表示激活备库成功标志，1 表示失败，0 表示成功
PROMOTE_STATUS=1

# 激活备库
su - $PG_OS_USER -c "pg_ctl promote -D /usr/local/postgresql_slave/pg_data/"
if [ $? -eq 0 ]; then
  echo -e "`date +%F\ %T`: `hostname` promote promote 执行成功. " >> $MONITOR_LOG
  PROMOTE_STATUS=0
fi
if [ $PROMOTE_STATUS -ne 0 ]; then
  echo -e "`date +%F\ %T`: promote promote 执行失败." >> $MONITOR_LOG
  return $PROMOTE_STATUS
fi
  echo -e "`date +%F\ %T`: pg_failover() function 调用成功." >> $MONITOR_LOG
  return 0
}
pg_failover
```

#### 1.4 指定keepalived.conf配置文件的route_id和vip地址

```shell
sed -i 's/HOSTNAME/'${HOSTNAME}'/g'  /usr/local/keepalived/etc/keepalived/keepalived.conf 

sed -i 's/vipaddress/10\.125\.26\.80/g'  /usr/local/keepalived/etc/keepalived/keepalived.conf

# 验证
cat /usr/local/keepalived/etc/keepalived/keepalived.conf
```

## 3、 创建依赖目录并创建软连接 👇

```shell
mkdir /usr/local/keepalived/temp/

```

## 4、修改启动文件 keepalived.service

vim /usr/lib/systemd/system/keepalived.service

```shell
[Unit]
Description=LVS and VRRP High Availability Monitor
After= network-online.target syslog.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/usr/local/keepalived/temp/keepalived.pid
KillMode=control-group
EnvironmentFile=-/usr/local/keepalived/etc/sysconfig/keepalived
ExecStart=/usr/local/keepalived/sbin/keepalived $KEEPALIVED_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

## 5、firewalld开启组播通信（注意按照网卡开通，此处是bond1）

```shell
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface bond1 --protocol vrrp -j ACCEPT

firewall-cmd --reload
```

## 6、启动服务

### 6.1 添加脚本执行权限

```shell
chmod 755 /usr/local/keepalived/bin/*.sh
```

### 6.2 设置开机自启动并启动服务

```shell
systemctl daemon-reload

# 开机自启动
systemctl enable keepalived

# 启动keepalived服务（由于咱们配置的是不抢占模式，注意需先停止所有节点的keepalived，然后优先启动master节点的keepalived服务）
systemctl start keepalived

systemctl status keepalived
```

# 