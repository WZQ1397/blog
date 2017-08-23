**1、安装依赖包**

```shell
yum -y install gcc gcc-c++ ncurses ncurses-devel cmake
```

**2、下载软件**

```shell
cd /usr/local
wget http://downloads.sourceforge.net/project/boost/boost/1.59.0/boost_1_59_0.tar.gz
wget http://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.14.tar.gz
```

**3、添加mysql用户和用户组**

```shell
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```

**4、解压安装mysql**

```shell
# 创建数据目录
mkdir -p /data/mysql

# 解压boost 
tar xzf boost_1_59_0.tar.gz

# 解压mysql，编译安装
tar xzf mysql-5.7.14.tar.gz

cd mysql-5.7.14
 cmake . \
 -DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
 -DMYSQL_DATADIR=/data/mysql \
 -DDOWNLOAD_BOOST=1 \
 -DWITH_BOOST=/usr/local/boost_1_59_0 \
 -DSYSCONFDIR=/etc \
 -DWITH_INNOBASE_STORAGE_ENGINE=1 \
 -DWITH_PARTITION_STORAGE_ENGINE=1 \
 -DWITH_FEDERATED_STORAGE_ENGINE=1 \
 -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
 -DWITH_MYISAM_STORAGE_ENGINE=1 \
 -DENABLED_LOCAL_INFILE=1 \
 -DENABLE_DTRACE=0 \
 -DDEFAULT_CHARSET=utf8mb4 \
 -DDEFAULT_COLLATION=utf8mb4_general_ci \
 -DWITH_EMBEDDED_SERVER=1


make
make install

# 复制到启动项
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
chmod +x /etc/init.d/mysql
chkconfig --add mysql
chkconfig mysql on
 
# 复制配置文件
cp /usr/local/mysql/support-files/my-default.cnf /etc/my.cnf

# 安装
/usr/local/mysql/bin/mysqld --initialize-insecure --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysql
```

**5、添加环境变量**

```shell
cd ~

vi .profile_bash

PATH=/usr/local/mysql/bin:$PATH

export PATH
```

**6、mysql主从复制（利用gtid特性**

```shell
# 关闭selinux和防火墙
systemctl stop firewalld.service
vi /etc/selinux/config

# 配置每个节点配置文件my.cnf
# 添加如下
log-bin=mysql-bin
server-id=3
gtid_mode=ON
log_slave_updates
enforce_gtid_consistency
（注：除了server-id每个节点不一样，其它都一样，也必须加，以一般主从复制区别，多了下面的3个参数）

# 主节点添加复制账号
grant replication slave  on *.* to 'backup'@'%'identified by'backup';

# 从节点执行
change master  to  master_host='192.168.10.140', master_user='backup', master_password='backup',master_port=3306,master_auto_position=1;
start slave;
```

**7、maxscale安装**

**安装依赖**

```shell
yum install libaio.x86_64 libaio-devel.x86_64 novacom-server.x86_64 libedit -y
```

**MaxScale 的下载地址：**

```shell
https://downloads.mariadb.com/files/MaxScale

rpm -ivh maxscale-beta-2.0.0-1.centos.7.x86_64.rpm
```

**在开始配置之前，需要在 master和backup 中为 MaxScale 创建两个用户，用于监控模块和路由模块。**

**创建监控用户**

```mysql
mysql> create user scalemon@'%' identified by "123456";

mysql> grant replication slave, replication client on *.* to scalemon@'%';
```

**创建路由用户**

```mysql
mysql> create user scaleroute@'%' identified by "123456";

mysql> grant select on mysql.* to scaleroute@'%';
```

**用户创建完成后，开始配置**

```shell
vi /etc/maxscale.cnf

[root@maxscale1 ~]# cat /etc/maxscale.cnf
# MaxScale documentation on GitHub:
# https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Documentation-Contents.md
 
# Global parameters
#
# Complete list of configuration options:
# https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Getting-Started/Configuration-Guide.md
 
[maxscale]
threads=1
log_info=1
logdir=/tmp/
 
# Server definitions
#
# Set the address of the server to the network
# address of a MySQL server.
#
 
[server1]
type=server
address=192.168.10.140
port=3306
protocol=MySQLBackend
 
[server2]
type=server
address=192.168.10.141
port=3306
protocol=MySQLBackend
 
[server3]
type=server
address=192.168.10.142
port=3306
protocol=MySQLBackend
 
# Monitor for the servers
#
# This will keep MaxScale aware of the state of the servers.
# MySQL Monitor documentation:
# https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Monitors/MySQL-Monitor.md
 
[MySQL Monitor]
type=monitor
module=mysqlmon
servers=server1,server2,server3
user=scalemon
passwd=123456
monitor_interval=10000
 
# Service definitions
#
# Service Definition for a read-only service and
# a read/write splitting service.
#
 
# ReadConnRoute documentation:
# https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Routers/ReadConnRoute.md
 
# [Read-Only Service]
# type=service
# router=readconnroute
# servers=server1
# user=myuser
# passwd=mypwd
# router_options=slave
 
# ReadWriteSplit documentation:
# https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Routers/ReadWriteSplit.md
 
[Read-Write Service]
type=service
router=readwritesplit
servers=server1,server2,server3
user=scaleroute
passwd=123456
max_slave_connections=100%
 
# This service enables the use of the MaxAdmin interface
# MaxScale administration guide:
# https://github.com/mariadb-corporation/MaxScale/blob/master/Documentation/Reference/MaxAdmin.md
 
[MaxAdmin Service]
type=service
router=cli
 
# Listener definitions for the services
#
# These listeners represent the ports the
# services will listen on.
#
 
# [Read-Only Listener]
# type=listener
# service=Read-Only Service
# protocol=MySQLClient
# port=4008
 
[Read-Write Listener]
type=listener
service=Read-Write Service
protocol=MySQLClient
port=4006
 
[MaxAdmin Listener]
type=listener
service=MaxAdmin Service
protocol=maxscaled
socket=default
```

**启动**

```sh
maxscale --config=/etc/maxscale.cnf
```

**查看**

**注意：这里用的是最新的2.0版本，登录已经不像老版本maxadmin –user=admin –password=mariadb这样了，而是像下面一样：**

```shell
[root@maxscale1 ~]# maxadmin -S /tmp/maxadmin.sock 
MaxScale> list services
Services.
--------------------------+----------------------+--------+---------------
Service Name              | Router Module        | #Users | Total Sessions
--------------------------+----------------------+--------+---------------
Read-Write Service        | readwritesplit       |      1 |     1
MaxAdmin Service          | cli                  |      2 |     4
--------------------------+----------------------+--------+---------------
命令也有变化，请自行FQ到Google查看
MaxScale> list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server             | Address         | Port  | Connections | Status              
-------------------+-----------------+-------+-------------+--------------------
server1            | 192.168.10.140  |  3306 |           0 | Master, Running
server2            | 192.168.10.141  |  3306 |           0 | Slave, Running
server3            | 192.168.10.142  |  3306 |           0 | Slave, Running
-------------------+-----------------+-------+-------------+--------------------
```

**测试**

在其它远程客户端连接maxscale所在服务器的数据库

```shell
mysql -h 192.168.10.140 -P4006 -uscalemon -p
mysql> select @@hostname;
+------------+
| @@hostname |
+------------+
| maxscale1  |
+------------+
1 row in set (0.00 sec)
 
关了一个slave后
MaxScale> list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server             | Address         | Port  | Connections | Status              
-------------------+-----------------+-------+-------------+--------------------
server1            | 192.168.10.140  |  3306 |           1 | Master, Running
server2            | 192.168.10.141  |  3306 |           1 | Slave, Running
server3            | 192.168.10.142  |  3306 |           0 | Down
-------------------+-----------------+-------+-------------+--------------------
```