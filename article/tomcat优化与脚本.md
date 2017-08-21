### 修改系统内核

修改linux的一些系统参数，以优化系统性能

- 修改LIMITS.CONF

```shell
$ vi /etc/security/limits.conf
# 增加
* soft nofile 65536
* hard nofile 65536
```

- 修改SYSCTL.CONF

```shell
# 备份
$ mv /etc/sysctl.conf /etc/sysctl.conf.bak

$ vi /etc/sysctl.conf
# 插入
net.ipv4.ip_forward = 0
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
kernel.sysrq = 0
kernel.core_uses_pid = 1
net.ipv4.tcp_syncookies = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 4096        87380   4194304
net.ipv4.tcp_wmem = 4096        16384   4194304
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 262144
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 30
net.ipv4.ip_local_port_range = 1024    65000
```

### 安装APR

- 下载apr-1.5.2.tar.gz和apr-util-1.5.4.tar.gz
  安装apr，方便tomcat 开启aoi 
  [apr下载地址](https://apr.apache.org/download.cgi)

```shell
$ tar -zxvf apr-1.5.2.tar.gz
$ tar -zxvf apr-util-1.5.4.tar.gz
$ yum install gcc gcc-c++ autoconf libtool

# apr安装
$ cd apr-1.5.2
$ ./configure --prefix=/usr/local/apr  
# --prefix=/usr/local/apr指apr的安装路径，以下要用到
$ make  
$ make install

# apr-util安装
$ cd apr-util-1.5.4
$ ./configure --with-apr=/usr/local/apr
# --with-apr=/usr/local/apr 指定APR安装路径
$ make
$ make install
```

### 安装JDK1.7

下载jdk-7u76-linux-x64.tar.gz 拷贝到/usr/local下面

```shell
cd /usr/local
$ tar -zxvf jdk-7u76-linux-x64.tar.gz

# mv 重命名 为jdk1.7.0_76
mv jdk1.7.0_76 jdk1.7

# 在环境变量/etc/profile里配置JAVA_HOME
JAVA_HOME=/usr/local/jdk1.7
PATH=$PATH:$HOME/bin:$JAVA_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME PATH CLASSPATH
 
# 环境变量立即生效
$ source .bash_profile
```

### 编译安装tomcat

#### 创建DATA目录

```
$ mkdir /data
```

#### 复制TOMCAT到目标目录并解压

```shell
$ cp apache-tomcat-7.0.64.tar.gz /data
$ tar -zxvf apache-tomcat-7.0.64.tar.gz
# mv 重命名为你所需的项目名称，如zhnx
$ mv apache-tomcat-7.0.64 zhnx
```

#### 进入TOMCAT精简目录

**请参照下面步骤一步一步顺序操作**

```shell
$ cd zach

# 把项目除了bin lib conf目录的其它全部删除
$ rm -rf LICENSE logs NOTICE RELEASE-NOTES RUNNING.txt  temp/webapps/work/

# 在同目录新建三个文件夹domain、server、sbin
$ mkdir domain
$ mkdir server
$ mkdir sbin

# 把bin 和lib目录移动到server下面
$ mv bin/ server/
$ mv lib/ server/

# 把conf目录移动到domain下面
$ mv conf/ domain/

# 安装tomcat native 开启aio
# 进入server/bin目录下安装tomcat native 与apr连起来 开启tomcat aio模式
$ cd server/bin

# 解压 tomcat native
$ tar -zxvf tomcat-native.tar.gz

# 进入安装tomcat  native目录
$ cd tomcat-native-1.1.33-src
$ cd jni
$ cd native

# 执行命令
$ ./configure --with-apr=/usr/local/apr --with-java-home=/usr/local/jdk1.7.0_76
$ make
$ make install

# 创建项目目录(这里项目名称示例CENTER）
$ cd /data/zach/domain
$ mkdir center

# 把conf目录移动到项目下
$ mv conf/ center/
```

**拷贝并修改脚本启动参数**

- **到目录/data/zhnx/sbin 下新建三个脚本，这里与center项目为例：startcenter.sh、stopcenter.sh和super.sh**

  https://github.com/WZQ1397/automatic-repo/tree/master/shell/tomcat

- 若有自己需要增加的jar就拷进去

$ cd /data/zach/server/lib 把jar考进去

### 项目部署

新建目录webapp名称与startcenter.sh里的webapps名称一样
日志名称与服务名一样
$ mkdir -p /data/zach/center_webapps

$ mkdir -p /data/zach/logs/center

center.war 部署至 center_webapps

部署时需将对应webapps目录下的文件及文件夹清空；
然后把项目war包拷入对应webapps目录下
启动服务

```
service center start
```