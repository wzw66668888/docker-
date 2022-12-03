**一、基于logstash filter功能将nginx默认的访问日志及error log转换为json格式并写入elasticsearch**

1.**在logstash 服务器编译安装nginx**

```shell
cd /usr/local/src && wget https://nginx.org/download/nginx-1.22.1.tar.gz

apt  install iproute2  ntpdate  tcpdump telnet traceroute nfs-kernel-server nfs-common  lrzsz tree  openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev   gcc openssh-server  iotop unzip zip

mkdir -p /apps/nginx 

./configure --prefix=/apps/nginx  make &&make install

cd /app/nginx && vim conf/nginx.conf

37    server_name  localhost www.nageedu.net;  (本地etc/hosts 添加域名解析)

vim html/index.html

 <h1> web1 magedu 20221128</h1>

./sbin/nginx
```

2.**编写logstash 日志收集配置**

cd /etc/logstash/conf.d    && vim nginxlog-to-es.conf

```shell
input {

 file {

  path => "/apps/nginx/logs/access.log"

  type => "nginx-accesslog"

  stat_interval => "1"

  start_position => "beginning"

 }



 file {

  path => "/apps/nginx/logs/error.log"

  type => "nginx-errorlog"

  stat_interval => "1"

  start_position => "beginning"

 }

}



filter {

 if [type] == "nginx-accesslog" {

 grok {

  match => { "message" => ["%{IPORHOST:clientip} - %{DATA:username} \[%{HTTPDATE:request-time}\] \"%{WORD:request-method} %{DATA:request-uri} HTTP/%{NUMBER:http_version}\" %{NUMBER:response_code} %{NUMBER:body_sent_bytes} \"%{DATA:referrer}\" \"%{DATA:useragent}\""] }

  remove_field => "message"

  add_field => { "project" => "magedu"}

 }

 mutate {

  convert => [ "[response_code]", "integer"]

  }

 }

 if [type] == "nginx-errorlog" {

  grok {

   match => { "message" => ["(?<timestamp>%{YEAR}[./]%{MONTHNUM}[./]%{MONTHDAY} %{TIME}) \[%{LOGLEVEL:loglevel}\] %{POSINT:pid}#%{NUMBER:threadid}\: \*%{NUMBER:connectionid} %{GREEDYDATA:message}, client: %{IPV4:clientip}, server: %{GREEDYDATA:server}, request: \"(?:%{WORD:request-method} %{NOTSPACE:request-uri}(?: HTTP/%{NUMBER:httpversion}))\", host: %{GREEDYDATA:domainname}"]}

   remove_field => "message"

  }

 }

}



output {

 if [type] == "nginx-accesslog" {

  elasticsearch {

   hosts => ["169.254.90.140:9200"]

   index => "magedu-nginx-accesslog-%{+yyyy.MM.dd}"

   user => "magedu"

   password => "123456"

 }}



 if [type] == "nginx-errorlog" {

  elasticsearch {

   hosts => ["169.254.90.140:9200"]

   index => "magedu-nginx-errorlog-%{+yyyy.MM.dd}"

   user => "magedu"

   password => "123456"

 }}

}
```

/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginxlog-to-es.conf  -t   检查文件格式

systemctl restart logstash.service   重启logstash 

登录kibana web页面创建并添加数据视图

![image-20221129151333367](C:\Users\wzw18\Desktop\极客时间\作业\图片\image-20221129151333367.png)

**二、基于logstash收集json格式的nginx访问日志**

**1.修改nginx的配置文件**

```shell
vim /apps/nginx/conf/nginx.conf

 25     #access_log  logs/access.log  main;
 26     log_format access_json '{"@timestamp":"$time_iso8601",'
 27       '"host":"$server_addr",'
 28       '"clientip":"$remote_addr",'
 29       '"size":$body_bytes_sent,'
 30       '"responsetime":$request_time,'
 31       '"upstreamtime":"$upstream_response_time",'
 32       '"upstreamhost":"$upstream_addr",'
 33       '"http_host":"$host",'
 34       '"uri":"$uri",'
 35       '"domain":"$host",'
 36       '"xff":"$http_x_forward_for",'
 37       '"referer":"$http_referer",'
 38       '"tcp_xff":"$proxy_protocol_addr",'
 39       '"http_user_agent":"$http_user_agent",'
 40       '"status":"$status"}';
 41     access_log /var/log/nginx/access.log access_json;
```

2.**编写logstash 日志收集配置**

vim /etc/logstash/conf.d/nginx-json-log-to-es.conf 

input {
  file {
    path => "/var/log/nginx/access.log"
    type => "nginx-json-accesslog"
    start_position => "end"
    stat_interval => "1"
    codec => json
  }
}

output {
  if [type] == "nginx-json-accesslog" {
    elasticsearch {
      hosts => ["169.254.90.140:9200"]
      index => "nginx-json-accesslog-2.107-%{+YYYY.MM.dd}"
      user => "magedu"
      password => "123456"
    }
  }
}

/usr/share/logstash/bin/logstash  -f  /etc/logstash/conf.d/nginx-json-log-to-es.conf  检查文件格式

systemctl restart logstash.service   重启logstash 

登录kibana web页面创建并添加数据视图

![屏幕截图 2022-11-29 180508](C:\Users\wzw18\Desktop\极客时间\作业\图片\屏幕截图 2022-11-29 180508.png)

**三、基于logstash收集java日志并实现多行合并**

环境准备：一台es 4c6G 两台es node 节点 2c4G

将logstash 上传至es master节点, 安装logstash-8.5.1-amd64.deb

```shell
 vim java-log-to-es.conf

input {
  file {
    path => "/data/eslogs/magedu-es-cluster1.log"
    type => "eslog"
    stat_interval => "5"
    start_position => "end"
    codec => multiline {
    pattern => "^\[[0-9]{4}\-[0-9]{2}\-[0-9]{2}"
    negate => "true"
    what => "previous"
  }
 }
}

output {
  if [type] == ["eslog"] {
    elasticsearch {
      hosts => ["169.254.90.137:9200"]
      index => "%{type}-%{+yyyy.ww}"
      user => "magedu"
      password => "123456"
}}
}
```

systemctl  restart  logstash.service  重启logstash 并到es 和kibana web页面验证

**四、基于logstash收集syslog类型日志(以haproxy替代网络设备)**

 环境准备：es集群(3个节点) 、logstash(1个)节点和一个haproxy节点

```shell
 apt install -y haproxy 

 vim /etc/haproxy/haproxy.cfg  #添加kibana 端口到haproxy后端节点

 36    listen kibana-5601
 37    bind 169.254.90.128:5601
 38    mode http
 39    log     global
 40    server  kibana-server1  169.254.90.140:5601 check inter 3s fall 3 rise 3
```

 systemctl restart haproxy.service  #重启后到web页面访问haproxy 的5601端口，查看haproxy日志

  修改rsyslog 配置文件并重启rsyslog

```shell
 vim /etc/rsyslog.d/49-haproxy.conf

  6 :programname, startswith, "haproxy" {
  7   #/var/log/haproxy.log      #注释
  8   stop
  9   @@169.254.90.128:514   #新增
 10 }

 systemctl restart rsyslog.service
```

 登录logstash 节点，编写logstash 日志收集配置

```shell
vim /etc/logstash/conf.d/network-syslog-to-es.conf

  input {
  syslog {
    host => "0.0.0.0"
    port => "514"
    type => "network-syslog"
   }
 }

output {
  if [type] == "network-syslog" {
    elasticsearch {
      hosts => ["169.254.90.140:9200"]
      index => "%{type}-%{+yyyy.MM.dd}"
      user => "magedu"
      password => "123456"
     }
   }
 }

 systemctl restart logstash.service   #重启logstash 并到es 和kibana web页面验证
```

**五、logstash收集日志并写入Redis、再通过其它logstash消费至elasticsearch并保持json格式日志的解析**

环境准备：es3台，logstash2台，redis一台

登录redis节点安装redis

```shell
 apt install redis -y

 vim /etc/redis/redis.conf

 69 bind 0.0.0.0

 507 requirepass 123456

 216 save ""                #关闭快照，减少磁盘io
 218 #save 900 1
 219 #save 300 10
 220 #save 60 10000
```

 systemctl restart redis-server.service

 redis-cli  #登录验证

 登录第一台logstash 节点，编写logstash收集nginx accesslog、errorlog到redis

```shell
 vim nginx-log-to-redis.conf

input {
  file {
    path => "/var/log/nginx/access.log"
    type => "magedu-nginx-accesslog"
    start_position => "beginning"
    stat_interval => "1"
    codec => "json" #对json格式日志进行json解析
  }

  file {
    path => "/apps/nginx/logs/error.log"
    type => "magedu-nginx-errorlog"
    start_position => "beginning"
    stat_interval => "1"
  }
}

filter {
  if [type] == "magedu-nginx-errorlog" {
    grok {
      match => { "message" => ["(?<timestamp>%{YEAR}[./]%{MONTHNUM}[./]%{MONTHDAY} %{TIME}) \[%{LOGLEVEL:loglevel}\] %{POSINT:pid}#%{NUMBER:threadid}\: \*%{NUMBER:connectionid} %{GREEDYDATA:message}, client: %{IPV4:clientip}, server: %{GREEDYDATA:server}, request: \"(?:%{WORD:request-method} %{NOTSPACE:request-uri}(?: HTTP/%{NUMBER:httpversion}))\", host: %{GREEDYDATA:domainname}"]}
      remove_field => "message" #删除源日志
    }
  }

output {
  if [type] == "magedu-nginx-accesslog" {
    redis {
      data_type => "list"
      key => "magedu-nginx-accesslog"
      host => "169.254.90.128"
      port => "6379"
      db => "0"
      password => "123456"
    }
  }
  if [type] == "magedu-nginx-errorlog" {
    redis {
      data_type => "list"
      key => "magedu-nginx-errorlog"
      host => "169.254.90.128"
      port => "6379"
      db => "0"
      password => "123456"
    }
  }
}
```

/usr/share/logstash/bin/logstash  -f  /etc/logstash/conf.d/nginx-log-to-redis.conf  -t   检查配置文件格式

systemctl restart logstash   #重启logstash 

登录redis节点验证数据是否写入

```shell
redis-cli  

  127.0.0.1:6379> AUTH 123456

  OK

  127.0.0.1:6379> KEYS *

1) "magedu-nginx-errorlog"

2) "magedu-nginx-accesslog
```

登录第二台logstash 编写logstash从redis收集日志到es集群

```shell
dpkg -i logstash-8.5.1-amd64.deb  

vim /etc/logstash/conf.d/redis-to-es.conf

input {
  redis {
  data_type => "list"
  key => "magedu-nginx-accesslog"
  host => "169.254.90.128"
  port => "6379"
  db => "0"
  password => "123456"
    }

  redis {
    data_type => "list"
    key => "magedu-nginx-errorlog"
    host => "169.254.90.128"
    port => "6379"
    db => "0"
    password => "123456"
    }
}

output {
  if [type] == "magedu-nginx-accesslog" {
    elasticsearch {
      hosts => ["169.254.90.142:9200"]
      index => "redis-magedu-nginx-accesslog-2.107-%{+YYYY.MM.dd}"
      user => "magedu"
      password => "123456"
    }
  }
  if [type] == "magedu-nginx-errorlog" {
    elasticsearch {
      hosts => ["169.254.90.142:9200"]
      index => "redis-magedu-nginx-errorlog-2.107-%{+YYYY.MM.dd}"
      user => "magedu"
      password => "123456"
    }
  }
}
```

/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/redis-to-es.conf  -t  检查配置文件格式

systemctl restart logstash   

登录es 和kibana web页面验证

**六、基于docker-compose部署单机版本ELK**

环境准备：一台4c8G 虚拟机或者云主机

1.准备docker和docker-compose 环境

tar xvf docker-20.10.19-binary-install.tar.gz

bash docker-install.sh 

2.安装git,创建git 工作目录

mkdir -p /data/es

git clone https://gitee.com/jiege-gitee/elk-docker-compose.git

3.内核参数优化

```shell
vim  /etc/sysctl.conf

net.ipv4.ip_forward=1
vm.max_map_count=262144
kernel.pid_max=4194303
fs.file-max=1000000
net.ipv4.tcp_max_tw_buckets=6000
net.netfilter.nf_conntrack_max=2097152

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0

vim /etc/security/limits.conf

*             soft    core            unlimited
*             hard    core            unlimited
*             soft    nproc           1000000
*             hard    nproc           1000000
*             soft    nofile          1000000
*             hard    nofile          1000000
*             soft    memlock         32000
*             hard    memlock         32000
*             soft    msgqueue        8192000
*             hard    msgqueue        8192000
```

reboot 重启确保参数生效

4.docker-compose up -d elasticsearch  #先拉起elasticsearch容器

5.docker exec -it elasticsearch /usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive \#设置elasticsearch默认账户密码  

6.检查logstash和kibana密码，确保和elasticsearch默认账户密码一致

7.docker-compose up -d  拉起logstash和kibana

8.echo "ERROR tcplog message1" > /dev/tcp/172.31.0.134/9889 #生成数据写入es

9.登录es和logstash web页面验证

![屏幕截图 2022-12-02 125743](C:\Users\wzw18\Desktop\屏幕截图 2022-12-02 125743.png)

**扩展**
    **1.大型的日志收集案例: filebeat-->logstash-->Redis<--logstash-->elasticsearch**

​     环境准备 3个节点es集群，两个logstash 节点，一个redis节点，一个nignx节点

​     登录nginx节点，安装filebeat插件

```shell
dpkg -i filebeat-8.5.1-amd64.deb

vim /etc/filebeat/filebeat.yml

15 filebeat.inputs:
22   - type: filestream 
25   id: magedu-app1
28   enabled: true
31   paths:
32     - /apps/nginx/logs/error.log
53   fields:
54   project: magedu
55   type: magedu-app1-errorlog
56 
57   - type: filestream
58   id: magedu-app1
59   enabled: true
60   paths:
61     - /var/log/nginx/access.log
62   fields:
63     project: magedu
64     type: magedu-app1-accesslog
65 
66   - type: filestream
67   id: magedu-app1
68   enabled: true
69   paths:
70     - /var/log/syslog
71   fields:
72     project: magedu
73     type: magedu-app1-syslog
170   output.logstash:
171   # The Logstash hosts
172   enabled: true
173   hosts: ["169.254.90.143:5044"]
174   loadbalance: true
175   worker: 1
176   compression_level: 3
```

  systemctl restart filebeat.service

  登录第一个logstash 节点，配置logstash配置文件接收filebeat日志并转发至redis

```shell
 vim /etc/logstash/conf.d/beats-magedu-to-redis.conf

  input { 
    beats {
      port => 5044
      codec => "json"
 }
 }

output {
  if [fields] [type]== "magedu-app1-accesslog" {
    redis {
      host => "169.254.90.128"
      password => "123456"
      port => "6379"
      db => "0"
      key => "magedu-app1-accesslog"
      data_type => "list"
   }
  }

  if [field] [types]== "magedu-app1-errorlog" {
    redis {
      host => "169.254.90.128"
      password => "123456"
      port => "6379"
      db => "0"
      key => "magedu-app1-errorlog"
      data_type => "list"
      }
    }

  if [field] [types] == "magedu-app1-syslog" {
    redis {
      host => "169.254.90.128"
      password => "123456"
      port => "6379"
      db => "0"
      key => "magedu-app1-syslog"
      data_type => "list"
      }
    }
  }
```

  /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/beats-magedu-to-redis.conf   -t  检查配置文件格式

  systemctl restart logstash  重启logstash并登录redis节点验证redis是否能查到数据

  登录第二台logstash 节点配置logstash节点从redis消费日志并写入elasticsearch

  vim magedu-filebeat-redis-to-es.conf

```shell
 input {
  redis {
    data_type => "list"
    key => "magedu-app1-accesslog"
    host => "169.254.90.128"
    port => "6379"
    db => "0"
    password => "123456"
    codec => "json"  
  }
  redis {
    data_type => "list"
    key => "magedu-app1-errorlog"
    host => "169.254.90.128"
    port => "6379"
    db => "0"
    password => "123456"
  }
   redis {
    data_type => "list"
    key => "magedu-app1-syslog"
    host => "169.254.90.128"
    port => "6379"
    db => "0"
    password => "123456"
  }
}

output {
  if [fields] [type] == "magedu-app1-accesslog" {
    elasticsearch {
      hosts => ["169.254.90.140:9200"]
      index => "magedu-app1-accesslog-%{+YYYY.MM.dd}"
      user => "magedu"
      password => "123456"
    }
  }

  if [fields] [type] == "magedu-app1-errorlog" {
    elasticsearch {
      hosts => ["169.254.90.140:9200"]
      index => "magedu-app1-errorlog-%{+YYYY.MM.dd}"
      user => "magedu"
      password => "123456"
    }
  }
  if [fields] [type] == "magedu-app1-syslog" {
    elasticsearch {
      hosts => ["169.254.90.140:9200"]
      index => "magedu-app1-syslog-%{+YYYY.MM.dd}"
      user => "magedu"
      password => "123456"
    }
  }
}
```

​    /usr/share/logstash/bin/logstash -f magedu-filebeat-redis-to-es.conf   -t  检查配置文件格式

​     systemctl restart logstash  重启logstash并登录es 和kibana web页面验证

   **2.日志写入数据库**

   环境准备:  环境准备 3个节点es集群，一个redis节点，一个数据库节点

   登录数据库节点，安装初始化数据库

```shell
apt install -y mariadb-server

   vim /etc/mysql/mariadb.conf.d/50-server.cnf

   28 bind-address            = 0.0.0.0  # 允许任何节点登录

   systemctl enable mariadb.service  --now

   mysql #登录数据库

   MariaDB [(none)]> create database elk character set utf8 collate utf8_bin;   #创建elk库

   MariaDB [(none)]> grant all privileges on elk.* to elk@"%" identified by '123456';  创建elk用户密码并授权

   MariaDB [(none)]> flush privileges;  # 刷新权限
```

   logstash 安装mysql-client客户端远程登录测试 

```shell
apt install -y mysql-client

   mysql -uelk -p123456 -h169.254.90.128
```

   登录logstash 节点安装logstash-output-jdbc插件

```shell
usr/share/logstash/bin/logstash-plugin install logstash-output-jdbc

   /usr/share/logstash/bin/logstash-plugin list | grep jdbc    #验证
```

   logsatsh节点下载并安装mysql-connector 软件包 output输出nginx日志到maridb

```shell
dpkg -c mysql-connector-j_8.0.31-1ubuntu22.04_all.deb

   dpkg -i mysql-connector-j_8.0.31-1ubuntu22.04_all.deb 

   mkdir -pv /usr/share/logstash/vendor/jar/jdbc

   cp /usr/share/java/mysql-connector-j-8.0.31.jar  /usr/share/logstash/vendor/jar/jdbc/

   chown logstash:logstash /usr/share/logstash/vendor/jar/ -R
```

   windows节点安装nvicat连接数据库，在elk库中创建eslog表并确认表头字段和类型

   登录logstash节点编写配置文件将nginx access 日志写入到db中

```shell
   vim jdbc-magedu-filebeat-redis-to-maridb.conf

   input {
    redis {
      data_type => "list"
      key => "magedu-nginx-accesslog"
      host => "169.254.90.128"
      port => "6379"
      db => "0" 
      password => "123456"
      codec => "json"  #json解析
  }

  redis {
    data_type => "list"
    key => "magedu-nginx-errorlog"
    host => "169.254.90.128"
    port => "6379"
    db => "0" 
    password => "123456"
  }
}

output {
  if [fields] [type] == "magedu-app1-accesslog" {
    elasticsearch {
      hosts => ["169.254.90.140:9200"]
      index => "jdbc-magedu-app1-accesslog-%{+YYYY.MM.dd}"
      user => "magedu"
      password => "123456"
   }
  }
  jdbc {
  connection_string => "jdbc:mysql://169.254.90.128/elk?user=elk&password=123456&useUnicode=true&characterEncoding=UTF8"
  statement =>  ["INSERT INTO elklog(clientip,size,responsetime,uri,http_user_agent,status) VALUES(?,?,?,?,?,?)", "clientip","size","responsetime","uri","http_user_agent","status"]
  }

  if [fields] [type] == "magedu-app1-errorlog" {
    elasticsearch {
      hosts => ["169.254.90.140:9200"]
      index => "jdbc-magedu-app1-errorlog-%{+YYYY.MM.dd}"
      user => "magedu"
      password => "123456"
    }
  }
}
```

   systemctl restart logstash  重启logstash  登录数据库验证数据是否写入

![屏幕截图 2022-12-01 222646](C:\Users\wzw18\Desktop\屏幕截图 2022-12-01 222646.png)

 **3.地图显示客户端IP城市**

  环境准备一个filebeat加nignx 节点，两个logstash 节点(其中一个logstash 版本为7.12)，一个redis节点，3个es节点

  安装logstash 7.12

  dpkg -i logstash-7.12.1-amd64.deb

   从官网下载GeoLite2-City_20221122.tar.gz 到/etc/logstash并解压

   配置logstash 7.12配置文件从redis消费数据发送到es集群

```shell
 vim magedu-filebeat-redis-to-es.conf 

input {
  redis {
    data_type => "list"
    key => "magedu-app1-accesslog"
    host => "169.254.90.128"
    port => "6379"
    db => "0"
    password => "123456"
    codec => "json"  #json解析
  }

  redis {
    data_type => "list"
    key => "magedu-app1-errorlog"
    host => "169.254.90.128"
    port => "6379"
    db => "0"
    password => "123456"
  }
}

filter {
  if [fields] [type] == "magedu-app1-accesslog" {
    geoip {
      source => "clientip"
      target => "geoip"
      database => "/etc/logstash/GeoLite2-City_20221122/GeoLite2-City.mmdb"

      add_field => "[ "[geoip][coordinates]", "%{[geoip][longitude]}" ]"
      add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
    }
    mutate {
    convert => [ "[geoip][coordinates]", "float"]
    }
  }
}

output {
  if [fields] [type] == "magedu-app1-accesslog" {
    elasticsearch {
      hosts => ["169.254.90.140:9200"]
      index => "logstash-magedu-app1-accesslog-%{+YYYY.MM.dd}"  #索引名称必须以logstash-开头
      user => "magedu"
      password => "123456"
    }
  }

  if [fields] [type] == "magedu-app1-errorlog" {
    elasticsearch {
      hosts => ["169.254.90.140:9200"]
      index => "jdbc-magedu-app1-errorlog-%{+YYYY.MM.dd}"
      user => "magedu"
      password => "123456"
    }
  }
}
```

 systemctl restart logstash.service  重启logstash服务

 登录nginx节点导入公网访问日志

```shell
  cd /var/log/nginx

  unzip blog-nginx_access.log.zip  

  cat blog-nginx_access.log >> access.log
```

  登录另外一台logstash 节点重启logstash收集日志发送到redis

  systemctl  restart  logstash.service

 登录kibana 节点创建数据视图并创建图像

 visualize library-->新建可视化-->Maps-->添加图层-->文档-->logstash-magedu-accesslog-->添加图   层-->保存并关闭



**4.logstash收集TCP日志**

​     登录logstash 节点，编写logstash 日志收集配置

```shell
  vim tcp-to-es.conf
  
input {
  tcp {
    port => 9889
    host => "169.254.90.141"
    type => "magedu-tcplog"
    mode => "server"
   }
  }

output {
  if [type] == "magedu-tcplog" {
    elasticsearch {
      hosts => ["169.254.90.142:9200"]
      index => "magedu-tcplog-%{+YYYY.MM.dd}"
      user => "magedu"
      password => "123456"
     }
    }
  }
```

```shell
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/tcp-to-es.conf  #检查配置文件格式

systemctl restart logstash.service   #重启logstash 
```

登录测试节点并安装navicat 

```shell
apt install netcat -y  

echo "tcp test message1" > /dev/tcp/169.254.90.141/9889

echo "tcp test message2" > /dev/tcp/169.254.90.141/9889

echo "nc test" | nc 169.254.90.141 9889
```

登录es 和kibana web页面验证

