## ELK日志分析
ELK 日志分析实例

一、ELK-web日志分析

二、ELK-MySQL 慢查询日志分析

三、ELK-SSH登陆日志分析

四、ELK-vsftpd 日志分析

五、ELK+RabbitMQ架构处理nginx及tomcat日志

六、ELK+filebeat处理MySQL慢查询日志

七、ELK+filebeat 处理mongodb日志

## 一、ELK-web日志分析

通过logstash grok正则将web日志过滤出来，输出到Elasticsearch 搜索引擎里，通过Kibana前端展示。

 ![1.png](http://www.178linux.com/ueditor/php/upload/image/20160602/1464839185717065.png)

**1.1、创建logstash grok 过滤规则**

```nginx
NGINXACCESS %{IPORHOST:remote_addr} – – \[%{HTTPDATE:time_local}\] "%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:httpversion}" %{INT:status} %{INT:body_bytes_sent} %{QS:http_referer} %{QS:http_user_agent}
```

**1.2、创建logstash web日志配置文件**

\#cat ./logstash/conf/ngx_log.conf

```json
input {
        file {
                type => "nginx_log"
                path => "/opt/nginx/logs/access.log"
        }
}  
filter {
  if [type] == "nginx_log" {
    grok {
      match => { "message" => "%{NGINXACCESS}" }
    }
    geoip {
      source => "remote_addr"
      target => "geoip"
      database => "/opt/logstash-2.0.0/conf/GeoLiteCity.dat"
      add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
      add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
    }
    mutate {
      convert => [ "[geoip][coordinates]","float", "body_bytes_sent","float", \
　　　　　　　　　　"body_bytes_sent.raw","float"]
    }
  }
}

output {
    stdout { codec => rubydebug }
    elasticsearch {
        hosts => "elk.test.com:9200"
        index => "ngx_log-%{+YYYY.MM}"
    }
}
```

**1.3、创建Kibana图形**

统计httpcode状态码

选择【Visualize】菜单，选择 【Pie chart】选项。字段选择status.raw，如下图所示:

![2.png](http://www.178linux.com/ueditor/php/upload/image/20160602/1464841431235759.png)

**统计访问前50 IP**

选择【Visualize】菜单，选择 【Vertical bar chart】选项。字段选择remote_addr.raw，如下图所示:

![3.png](http://www.178linux.com/ueditor/php/upload/image/20160602/1464841459364774.png)

**统计 403-405 状态码**

选择【Visualize】菜单，选择 【Line chart】选项。字段选择status.raw，如下图所示:

![4.png](http://www.178linux.com/ueditor/php/upload/image/20160602/1464841491850854.png)

其它图形统计，就不详细举例了。

**详细图形展示如下:**

![5.png](http://www.178linux.com/ueditor/php/upload/image/20160602/1464841538310714.png)

![6.png](http://www.178linux.com/ueditor/php/upload/image/20160602/1464841541866268.png)

![7.png](http://www.178linux.com/ueditor/php/upload/image/20160602/1464841544471220.png)

## 二、ELK-MySQL 慢查询日志分析

**2.1、创建logstash grok 过滤规则**

\#cat ./logstahs/patterns/mysql_slow

```mysql
MYSQLSLOW "# User@Host: %{WORD:user}\[%{WORD}\] @ (%{HOST:client_hostname}|) \[(%{IP:client_ip}|)\]",
"# Thread_id: %{NUMBER:thread_id:int} \s*Schema: (%{WORD:schema}| ) \s*Last_errno: \
%{NUMBER:last_errno:int} \s*Killed: %{NUMBER:killed:int}",
"# Query_time: %{NUMBER:query_time:float} \s*Lock_time: %{NUMBER:lock_time:float} \
\s*Rows_sent: %{NUMBER:rows_sent:int} \s*Rows_examined: %{NUMBER:rows_examined:int}",
"# Bytes_sent: %{NUMBER:bytes_sent:int}",
"(?m)SET timestamp=%{NUMBER:timestamp};%{GREEDYDATA:mysql_query}"
```

**2.2、创建logstash MySQL-Slow慢查询配置文件**

\#cat ./logstash/conf/MySQL-Slow.conf

```json
input {
  file {
    type => "mysql-slow"
    path => "/var/log/mysql_slow_log.log"  
  }
}  
filter {
if [type] == "mysql-slow" {  
multiline {
    pattern => "^#|^SET"
    negate => true
    what => "previous"
}  
grok {
    match => { "message" => "%{MYSQLSLOW}"  }
}
mutate {
         gsub => [ "mysql_query", "\n", " " ]
         gsub => [ "mysql_query", "  ", " " ]
         add_tag => "mutated_mysql_query"
}
multiline {
    pattern => "(# User|# Thread|# Query|# Time|# Bytes)"
    negate => false
    what => "next"
}
date {
    match => [ "timestamp","UNIX" ]
}
mutate {
    remove_field => [ "timestamp" ]
}
}
}  
output {
    stdout { codec => rubydebug }
    elasticsearch {
        hosts => "elk.test.com:9200"
        index => "mysql_slow_log-%{+YYYY.MM}"
    }
}
```

**2.3、详细图形展示如下:**

**![8.png](http://www.178linux.com/ueditor/php/upload/image/20160602/1464841994645026.png)**

## 三、ELK-SSH登陆日志分析

**3.1、创建logstash grok 过滤规则**

\#cat ./logstahs/patterns/ssh
```sh
SECURELOG %{WORD:program}\[%{DATA:pid}\]: %{WORD:status} password for ?(invalid user)? %{WORD:USER} from %{DATA:IP} port

SYSLOGPAMSESSION %{SYSLOGBASE} (?=%{GREEDYDATA:message})%{WORD:pam_module}\(%{DATA:pam_caller}\): session %{WORD:pam_session_state} for user %{USERNAME:username}(?: by %{GREEDYDATA:pam_by})?

SYSLOGBASE2 (?:%{SYSLOGTIMESTAMP:timestamp}|%{TIMESTAMP_ISO8601:timestamp8601}) (?:%{SYSLOGFACILITY} )?%{SYSLOGHOST:logsource} %{SYSLOGPROG}:
```
****

**3.2、创建logstash ssh配置文件**

\#cat ./logstash/conf/ssh.conf

```json
input {
    file {
        type => "seclog"
        path => "/var/log/secure"
   }
}
filter {
if [type] == "seclog" {
    grok {
        match => { "message" => "%{SYSLOGPAMSESSION}" }
        match => { "message" => "%{SECURELOG}" }
        match => { "message" => "%{SYSLOGBASE2}" }
    }
    geoip {
        source => "IP"
        fields => ["city_name"]
        database => "/opt/logstash-2.0.0/conf/GeoLiteCity.dat"
    }
    if ([status] == "Accepted") {
        mutate {
        add_tag => ["Success"]
        }
    }
    else if ([status] == "Failed") {
        mutate {
        add_tag => ["Failed"]
        }
    }
}
output {
    stdout { codec => rubydebug }
    elasticsearch {
        hosts => "elk.test.com:9200"
        index => "sshd_log-%{+YYYY.MM}"
    }
}
```

PS:添加状态标签，便于Kibana 统计

```json
if ([status] == "Accepted") {              #判断字段[status]值，匹配[Accepted]
        mutate {
        add_tag => ["Success"]      #添加标签[Success]
        }
}
else if ([status] == "Failed") {           #判断字段[status]值，匹配[Failed]
        mutate {
        add_tag => ["Failed"]       #添加标签[Failed]
        }
}
```

****

**3.3、详细图形展示如下:**

**![9.png](http://www.178linux.com/ueditor/php/upload/image/20160602/1464842162381708.png)**

## 四、ELK-vsftpd 日志分析

****

4.1、创建logstash grok 过滤规则

\#cat ./logstahs/patterns/vsftpd
```
VSFTPDCONNECT \[pid %{WORD:pid}\] %{WORD:action}: Client \"%{DATA:IP}\"
VSFTPDLOGIN \[pid %{WORD:pid}\] \[%{WORD:user}\] %{WORD:status} %{WORD:action}: Client \"%{DATA:IP}\"VSFTPDACTION \[pid %{DATA:pid}\] \[%{DATA:user}\] %{WORD:status} %{WORD:action}: Client \"%{DATA:IP}\", \"%{DATA:file}\", %{DATA:bytes} bytes, %{DATA:Kbyte_sec}Kbyte/sec 
```
4.2、创建logstash vsftpd配置文件

\#cat ./logstash/conf/vsftpd.conf

```json
input {
    file {
        type => "vsftpd_log"
        path => "/var/log/vsftpd.log"
    }
}
filter {
    if [type] == "vsftpd_log" {
        grok {
            match => { "message" => "%{VSFTPDACTION}" }
            match => { "message" => "%{VSFTPDLOGIN}" }
            match => { "message" => "%{VSFTPDCONNECT}" }
        }
    }
}
output {
    stdout { codec => rubydebug }
    elasticsearch {
        hosts => "elk.test.com:9200"
        index => "vsftpd_log-%{+YYYY.MM}"
    }
}
```

**4.3、详细图形展示如下:**

****

![10.png](http://www.178linux.com/ueditor/php/upload/image/20160602/1464842264820223.png)



## 五、ELK+RabbitMQ处理日志

![img](http://www.178linux.com/ueditor/php/upload/image/20160729/1469789491548352.png)

#### RabbitMQ配置

安装RabbitMQ

```shell
yum install rabbitmq-server
```

启用RabbitMQ的web管理功能

```
/usr/lib/rabbitmq/bin/rabbitmq-plugins enable rabbitmq_management
/usr/lib/rabbitmq/bin/rabbitmq-plugins list
```

下载并安装命令管理工具rabbitmqadmin

```shell
wget http://rabbitmq-server:15672/cli/rabbitmqadmin
mv rabbitmqadmin /usr/local/bin
chmod +x /usr/local/bin/rabbitmqadmin
```

给rabbitmqadmin工具准备配置文件

```mysql
# vim /etc/mqadmin.conf
[default]
hostname = localhost
port = 55672
username = liang
password = liang123
```

创建一个vhost和user并赋权

```shell
rabbitmqctl add_user liang liang123
rabbitmqctl add_vhost elk
rabbitmqctl set_permissions -p elk liang ".*" ".*" ".*"
rabbitmqctl set_user_tags liang administrator
rabbitmqctl list_permissions -p elk
```

创建一个exchange

```shell
rabbitmqadmin -c /etc/mqadmin.conf declare exchange --vhost=elk name=elk_exchange type=direct
```

创建一个queue

```shell
rabbitmqadmin -c /etc/mqadmin.conf declare queue --vhost=elk name=elk_queue durable=true
```

创建一个binding，绑定之前创建的exchange和queue并设置一个routing_key

```shell
rabbitmqadmin -c /etc/mqadmin.conf --vhost=elk declare binding source="elk_exchange" destination="elk_queue" routing_key="elk_key"
```

以上关于RabbitMQ的配置均可以通过登录web控制台进行操作

#### Logstash配置

配置nginx服务器输出json格式日志

```nginx
log_format json '{"@timestamp":"$time_iso8601",'
               '"host":"$server_addr",'
               '"clientip":"$remote_addr",' '"size":$body_bytes_sent,'
               '"responsetime":$request_time,'
               '"upstreamtime":"$upstream_response_time",'
               '"upstreamhost":"$upstream_addr",'
               '"http_host":"$host",'
               '"url":"$uri",'
               '"xff":"$http_x_forwarded_for",'
               '"referer":"$http_referer",'
               '"agent":"$http_user_agent",'
               '"status":"$status"}';
access_log  /usr/local/nginx/logs/api_json.log  json;
```

配置logstash agent采集nginx日志并输出到RabbitMQ；为了排错，同时输出一份日志到本地。

```json
# vim /etc/logstash/conf.d/ngx_log.conf
input {
    file {
	path => "/usr/local/nginx/logs/api_json.log"
	codec => "json"
	type => "nginx"
    }
}

output {
    rabbitmq { 
	host => "RabbitMQ_server"
	port => "5672"
	vhost => "elk"
	exchange => "elk_exchange"
	exchange_type => "direct"
	key => "elk_key"
        user => "liang"
        password => "liang123"
	}
    stdout { codec => rubydebug }
    }
```

配置tomcat服务器输出json格式日志，修改工程的logback.xml配置文件，添加如下配置

```java
	<appender name="LOGSTASH" class="ch.qos.logback.core.rolling.RollingFileAppender"> 
        <file>${catalina.base}/logs/tomcat_json.log</file>
	    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
	        <charset>utf8</charset>
	    </encoder> 
	    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">  
	        <fileNamePattern>${catalina.base}/logs/tomcat_json-%d{yyyy-MM-dd}.log</fileNamePattern>  
	    </rollingPolicy>  
	</appender>
    <root level="info">
        <appender-ref ref="LOGSTASH" />
    </root>
```

下载依赖的jar包logstash-logback-encoder到{CATALINA_BASE}/lib

```
wget http://central.maven.org/maven2/net/logstash/logback/logstash-logback-encoder/4.4/logstash-logback-encoder-4.4.jar
```

配置logstash agent采集tomcat日志并输出到RabbitMQ

```json
# vim /etc/logstash/conf.d/tomcat_log.conf
input {
    file {
	path => "/usr/local/tomcat/logs/tomcat_json.log"
	codec => "json"
	type => "tomcat"
    }
}

output {
    rabbitmq { 
	host => "RabbitMQ_server"
	port => "5672"
	vhost => "elk"
	exchange => "elk_exchange"
	exchange_type => "direct"
	key => "elk_key"
        user => "liang"
        password => "liang123"
	}
    stdout { codec => rubydebug }
    }
```

配置logstash indexer把日志从RabbitMQ输出到Elasticsearch

```json
# vim /etc/logstash/conf.d/rabbitmq.conf
input {
    rabbitmq {
	host => "127.0.0.1"
	subscription_retry_interval_seconds => "5"
	vhost => "elk"
	exchange => "elk_exchange"
	queue => "elk_queue"
	durable => "true"
	key => "elk_key"
	user => "liang"
	password => "liang123"
    }
}
output {
    if [type] == "nginx" {
        elasticsearch {
            hosts => "Elasticsearch_server:9200"
            user => "logstash"
            password => "123456"
	    index => "nginx-%{+YYYY.MM.dd}"
        }
    }
    else if [type] == "tomcat" {
        elasticsearch {
            hosts => "Elasticsearch_server:9200"
            user => "logstash"
            password => "123456"
	    index => "tomcat-%{+YYYY.MM.dd}"
        }
    }
    else {
	file {
	    path => "/var/log/logstash/unknown_messages.log"
	    } 
	}
    stdout { codec => rubydebug }
}
```

启动logstash服务

```
service logstash start
```

在RabbitMQ服务器上查看是否接收到日志消息，登录RabbitMQ的web控制台查看详细信息。

```
rabbitmqctl list_queues -p elk
```

**其他安装配置不再赘述**



## 六、ELK+filebeat处理MySQL慢查询日志

#### **filebeat配置**

```yaml
filebeat:
  prospectors:
    -
      paths:
        - /var/logs/mysql/slow.log
      document_type: mysqlslowlog
      input_type: log
      multiline:
        negate: true
        match: after
  registry_file: /var/lib/filebeat/registry
output:
  logstash:
    hosts: ["10.6.66.14:5046"]
shipper:
logging:
  files:
```

#### **logstash配置**

```json
# vi /etc/logstash/conf.d/01-beats-mysql-input.conf 
input {
  beats {
    port => 5046
    host => "10.6.66.14"
 }
}
filter {
  if [type] == "mysqlslowlog" {
  grok {
    match => { "message" => "(?m)^#\s+User@Host:\s+%{USER:user}\[[^\]]+\]\s+@\s+(?:(?<clienthost>\S*) )?\[(?:%{IPV4:clientip})?\]\s+Id:\s+%{NUMBER:row_id:int}\n#\s+Query_time:\s+%{NUMBER:query_time:float}\s+Lock_time:\s+%{NUMBER:lock_time:float}\s+Rows_sent:\s+%{NUMBER:rows_sent:int}\s+Rows_examined:\s+%{NUMBER:rows_examined:int}\n\s*(?:use %{DATA:database};\s*\n)?SET\s+timestamp=%{NUMBER:timestamp};\n\s*(?<sql>(?<action>\w+)\b.*;)\s*(?:\n#\s+Time)?.*$" }
  }
    date {
      match => [ "timestamp", "UNIX", "YYYY-MM-dd HH:mm:ss"]
      remove_field => [ "timestamp" ]
    }
  }
}
```

**output段配置**

```json
# vi /etc/logstash/conf.d/30-beats-mysql-output.conf 
output {
    if "_grokparsefailure" in [tags] {
      file { path => "/var/log/logstash/grokparsefailure-%{[type]}-%{+YYYY.MM.dd}.log" }
    }
 
  if [@metadata][type] in [ "mysqlslowlog" ] {
    elasticsearch {
      hosts => ["10.6.66.14:9200"]
      sniffing => true
      manage_template => false
      template_overwrite => true
      index => "%{[@metadata][beat]}-%{[type]}-%{+YYYY.MM.dd}"
      document_type => "%{[@metadata][type]}"
    }
  }
}
```



## 七、ELK+filebeat 处理mongodb日志

从3.0版本开始，mongodb日志内容包含severity level和component

> \<timestamp\> \<severity\> \<component\> [\<context\>] \<message\>

#### **filebeat配置**

```yaml
# vi /etc/filebeat/filebeat.yml
filebeat:
  prospectors:
    -
      paths:
        - /www.ttlsa.com/logs/mysql/slow.log
      document_type: mysqlslowlog
      input_type: log
      multiline:
        negate: true
        match: after
    -
      paths:
        - /www.ttlsa.com/logs/mongodb/mongodb.log 
      document_type: mongodblog
  registry_file: /var/lib/filebeat/registry
output:
  logstash:
    hosts: ["10.6.66.18:5046"]
shipper:
logging:
  files:
```

#### **logstash配置**

```json
# vi /etc/logstash/conf.d/01-beats-mysql-input.conf 
input {
  beats {
    port => 5046
    host => "10.6.66.14"
 }
}
filter {
  if [type] == "mongodblog" {
 
    grok {
       match => ["message","%{TIMESTAMP_ISO8601:timestamp}\s+%{MONGO3_SEVERITY:severity}\s+%{MONGO3_COMPONENT:component}\s+(?:\[%{DATA:context}\])?\s+%{GREEDYDATA:body}"]
    }
 
    if [body] =~ "ms$"  {  
       grok {
         match => ["body","query\s+%{WORD:db_name}\.%{WORD:collection_name}.*}.*\}(\s+%{NUMBER:spend_time:int}ms$)?"]
       }
    }
 
    date {
      match => [ "timestamp", "UNIX", "YYYY-MM-dd HH:mm:ss", "ISO8601"]
      remove_field => [ "timestamp" ]
    }
  }
}
```

**output段配置**

```json
# vim 30-beats-output.conf 
output {
    if "_grokparsefailure" in [tags] {
      file { path => "/var/log/logstash/grokparsefailure-%{[type]}-%{+YYYY.MM.dd}.log" }
    }
 
if [@metadata][type] in [ "mysqlslowlog", "mongodblog" ] {
    elasticsearch {
      hosts => ["10.6.66.18:9200"]
      sniffing => true
      manage_template => false
      template_overwrite => true
      index => "%{[@metadata][beat]}-%{[type]}-%{+YYYY.MM.dd}"
      document_type => "%{[@metadata][type]}"
    }
}
```
