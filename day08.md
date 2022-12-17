**一、基于docker-compose或二进制部署skywalking**

**skywalking二进制安装**

1.部署单节点es

```shell
vim /etc/sysctl.conf

  net.ipv4.ip_forward = 1

  vm.max_map_count=262144
```

安装elasticsearch-8.5.1-amd64.deb

```shell
dpkg -i elasticsearch-8.5.1-amd64.deb

vim /etc/elasticsearch/elasticsearch.yml

  17  cluster.name: skaywalking-es

  23 node.name: node1

  33 path.data: /var/lib/elasticsearch

  37 path.logs: /var/log/elasticsearch

  56 network.host: 0.0.0.0

  61 http.port: 9200

  70 discovery.seed_hosts: ["169.254.90.147",]

  74 cluster.initial_master_nodes: ["169.254.90.147",]

  98 xpack.security.enabled: false
  100 xpack.security.enrollment.enabled: false

  103 xpack.security.http.ssl:
  104   enabled: false
  105   keystore.path: certs/http.p12

  108 xpack.security.transport.ssl:
  109   enabled: false
  110   verification_mode: certificate
  111   keystore.path: certs/transport.p12
  112   truststore.path: certs/transport.p12

  119 http.host: 0.0.0.0

  systemctl enable elasticsearch.service   --now
```

2.部署skywalking

```shell
apt install -y openjdk-11-jdk  安装jdk环境

tar -xvf apache-skywalking-apm-9.3.0.tar.gz 

ln -sv /apps/apache-skywalking-apm-bin /apps/skywalking

vim config/application.yml

vim  /apps/skywalking/config/application.yml

132 storage:
133   selector: ${SW_STORAGE:elasticsearch}       #修改skaywalking 后端存储为es

136     clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:169.254.90.147:9200}   #填写es单节点地址

vim /etc/systemd/system/skaywalking.service

[Unit]
Description=Apache Skywalking
After=network.target
[Service]
Type=oneshot
User=root
WorkingDirectory=/apps/skywalking/bin/
ExecStart=/bin/bash /apps/skywalking/bin/startup.sh
RemainAfterExit=yes
RestartSec=5
[Install]
WantedBy=multi-user.target

systemctl daemon-reload 

systemctl enable skaywalking --now

tail -f logs/*.log   #查看skaywalking 日志确保无error
```

登录skaywalking前端验证:
![image-20221214160324042](https://user-images.githubusercontent.com/82318011/208225770-0a33bb87-242d-4adb-94eb-7279ec79241a.png)



**二、实现单体服务halo博客和jenkins的请求链路跟踪**

**SkyWalking-java博客追踪**

安装SkyWalking-agent

```shell
apt install openjdk-11-jdk  #安装jdk环境

java -version   #查看java安装版本

mkdir /data && cd /data

tar xvf apache-skywalking-java-agent-8.13.0.tgz

vim /data/skywalking-agent/config/agent.config

20 agent.service_name=${SW_AGENT_NAME:magedu-halo}   #skaywalking前期ui界面显示的项目名称

23 agent.namespace=${SW_AGENT_NAMESPACE:magedu}

101 collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:169.254.90.146:11800}  #填写skaywalking-server地址


```

安装halo博客

```shell
mkdir /apps && cd /apps/

wget https://dl.halo.run/release/halo-1.6.1.jar

#绝对路径启动SkyWalking-javaagent和halo博客

java -javaagent:/data/skywalking-agent/skywalking-agent.jar -jar /apps/halo-1.6.1.jar

#生产环境启动命令示例：

 java -javaagent:/skywalking-agent/skywalking-agent.jar  -DSW_AGENT_NAMESPACE=xyz \

-DSW_AGENT_NAME=abc-application \

-Dskywalking.collector.backend_service=skywalking.abc.xyz.com:11800 \

-jar abc-xyz-1.0-SNAPSHOT.jar
```

halo登录界面初始化并登录，编写文章产生访问信息
![image-20221215120056117](https://user-images.githubusercontent.com/82318011/208225782-56ef1911-0b21-457c-8965-c1cf67413b7c.png)


skywalking 验证数据
![image-20221215120126350](https://user-images.githubusercontent.com/82318011/208225793-336fb313-962c-4c84-915e-80f3d61573f8.png)



**jenkins的请求链路跟踪**

安装SkyWalking-agent

```shell
apt install openjdk-11-jdk  #安装jdk环境

java -version   #查看java安装版本

mkdir /data && cd /data

tar xvf apache-skywalking-java-agent-8.13.0.tgz

vim /data/skywalking-agent/config/agent.config

20 agent.service_name=${SW_AGENT_NAME:magedu-jenkens}

23 agent.namespace=${SW_AGENT_NAMESPACE:magedu}

101 collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:169.254.90.146:11800}
```

安装tomcat

```shell
mkdir /apps/ && cd /apps

wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.84/bin/apache-tomcat-8.5.84.tar.gz

tar xvf apache-tomcat-8.5.84.tar.gz

vim /apps/apache-tomcat-8.5.84/bin/catalina.sh

125 CATALINA_OPTS="$CATALINA_OPTS -javaagent:/data/skywalking-agent/skywalking-agent.jar"; export CATALINA_OPTS   # 配置启动catalina.sh脚本时， skywalking-agent可以同时启动，跟踪jenkins数据

#上传jenkins.war包替换tomcat webapps源目录

/apps/apache-tomcat-8.5.84/bin/catalina.sh run   #运行catalina脚本，查看启动过程无报错
```

登录jenkins 8080 端口验证
![image-20221215154819131](https://user-images.githubusercontent.com/82318011/208225872-8cff0700-725e-498b-94f2-90d891e5fb61.png)


登录skywalking 前端验证
![image-20221215154916201](https://user-images.githubusercontent.com/82318011/208225876-ff94e37e-ab90-4bbe-a3be-021327788fd2.png)


**三、实现dubbo微服务实现链路跟踪案例**

部署注册中心(169.254.90.144)

```shell
apt install openjdk-8-jdk

java -version  #查看java版本

mkdir /apps && cd /apps

wget https://dlcdn.apache.org/zookeeper/zookeeper-3.7.1/apache-zookeeper-3.7.1-bin.tar.gz

tar xvf apache-zookeeper-3.7.1-bin.tar.gz

cp /apps/apache-zookeeper-3.7.1-bin/conf/zoo_sample.cfg /apps/apache-zookeeper-3.7.1- 

bin/conf/zoo.cfg    #修改模板文件名称为zoo.cfg

/apps/apache-zookeeper-3.7.1-bin/bin/zkServer.sh start   #启动zk

lsof -i:2181  #查看zk 2181 端口是否监听
```



部署provider(169.254.90.148)

```shell
apt install openjdk-8-jdk -y

mkdir /data && cd /data

tar xvf apache-skywalking-java-agent-8.13.0.tgz   #安装skywalking-agent

vim /data/skywalking-agent/config/agent.config

20 agent.service_name=${SW_AGENT_NAME:magedu-dubbo-server1}  #agent-server名称为magedu-dubbo-server1

23 agent.namespace=${SW_AGENT_NAMESPACE:magedu2}  #agent命名空间

101collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:skywalking.example.com:11800}   # 将skywalking-server Ip地址替换为域名

# 添加主机名解析和环境变量

echo "169.254.90.146  skywalking.example.com"  >> /etc/hosts

echo "export ZK_SERVER1=169.254.90.144"  >> /etc/profile

source /etc/profile  

echo $ZK_SERVER1  #测试变量是否生效

mkdir -pv /apps/dubbo/provider  

# 启动dubbo-server端

java -javaagent:/data/skywalking-agent/skywalking-agent.jar -jar /apps/dubbo/provider/dubbo-server.jar
```

部署consumer(169.254.90.145)

```shell
apt install openjdk-8-jdk -y

mkdir /data && cd /data

tar xvf apache-skywalking-java-agent-8.13.0.tgz

vim /data/skywalking-agent/config/agent.config

20  agent.service_name=${SW_AGENT_NAME:magedu-dubbo-consumer}   #agent-server名称为magedu-dubbo-consumer

23 agent.namespace=${SW_AGENT_NAMESPACE:myserver}

101 collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:${SW_SERVER}:11800}

#添加主机名解析与环境变量

vim /etc/profile

export SW_SERVER="169.254.90.146"    #skywalking-server地址
export ZK_SERVER1="169.254.90.144"   #skywalking-zk 注册中心地址

source /etc/profile

mkdir -pv /apps/dubbo/consumer

# 启动dubbo-server端

java -javaagent:/data/skywalking-agent/skywalking-agent.jar -jar /apps/dubbo/consumer/dubbo-client.jar
```

浏览器访问consumer:8080/hello?name=zhangsan
![image-20221215185832413](https://user-images.githubusercontent.com/82318011/208225895-13b9a1c8-ae3b-4c4d-a557-809d06f3e319.png)

![image-20221215190151342](https://user-images.githubusercontent.com/82318011/208225901-4af67e57-2fc4-4e94-90a4-932c158a5d68.png)

![image-20221215190218506](https://user-images.githubusercontent.com/82318011/208225915-d82a3dca-f3f5-46ba-b7cd-0d406f47d1d5.png)



验证skywalking数据
![image-20221215190034740](https://user-images.githubusercontent.com/82318011/208225898-cc23ac65-6ad9-4427-b5cd-8ecb656d2536.png)




**四、实现skywalking的钉钉告警**

skywalking 告警-指标：

service_resp_time #服务的响应时间

service_sla #服务的http请求成功率SLA,比如99%等。

service_cpm #表示每分钟的吞吐量

service_apdex : 应用性能指数是0.8是0.x

service_percentile: 指定最近多少数据范围内的响应时间百分比,即p99, p95, p90, p75, p50在内的数据统计结果

endpoint_relation_cpm #端点的每分钟的吞吐量

endpoint_relation_resp_time #端点的响应时间

endpoint_relation_sla #端点的http请求成功率SLA,比如99%等

endpoint_relation_percentile #端点的最近多少数据范围内的响应时间百分比,即p99、p95、p90、p75、p50在内的数据统计结果

 vim /apps/skywalking/config/alarm-settings.yml   #替换skaywalking默认告警配置

rules: #定义rule规则
  service_cpm_rule: #唯一的规则名称,必须以_rule结尾
    # Metrics value need to be long, double or int
    metrics-name: service_cpm #指标名称
    op: ">" #操作符,>, >=, <, <=,==threshold: 1 #指标阈值
    # The length of time to evaluate the metrics
    period: 1 #评估指标的间隔周期
    # How many times after the metrics match the condition, will trigger alarm
    count: 1 #匹配成功多少次就会触发告警
    # How many times of checks, the alarm keeps silence after alarm triggered, default as same as period. 
    #silence-period: 3
    silence-period: 0 #触发告警后的静默时间
    message: "服务dubbo-provider service_cpm 被访问的次数超过1次了" #告警信息

dingtalkHooks:
  textTemplate: |-
    {
      "msgtype": "text",
      "text": {
        "content": "Apache SkyWalking Alarm: \n %s."
      }
    }
  webhooks: - url: https://oapi.dingtalk.com/robot/send?access_token=0c2d4acae94248658bcd386b1d9d7599ce89dab64e99b67702a07b540af6f   #注意钉钉关键字

systemctl restart skywalking.service  #重启skywalking 

验证：浏览器访问dubbo client   http://169.254.90.145:8080/hello?name=lilei  等待dingding发送告警



**扩展**
**1.实现python Django项目的请求链路跟踪**

环境准备Ubuntu20.04/python3.8，2c4g 虚拟机或云主机

```shell
apt install python3-pip  

pip3 install "apache-skywalking"
```

```python
python3

>>> from skywalking import agent, config

>>> config.init(collector_address='169.254.90.148:11800' , service_name='python-app') #测试注册

>>> agent.start()  #启动agent并到skywalking前端页面验证是否python-app 服务
```

![image-20221216141059270](https://user-images.githubusercontent.com/82318011/208226039-3c6766ad-e958-464d-a6ac-eb05fe28fbcf.png)


```shell
mkdir /apps/  && cd /apps

tar xvf django-test.tgz

cd django-test/

pip3 install -r requirements.txt  #安装依赖模块

django-admin startproject mysite  #创建django项目mysite

cd mysite/  && python3 manage.py startapp myapp  #创建myapp项目

python3 manage.py startapp myapp
```

```shell
#初始化数据库

python3 manage.py makemigrations

python3 manage.py migrate

#创建项目管理员，用于登录web界面

python3 manage.py createsuperuser  #默认用户名为root  密码不要太简单

#在python django 声明环境变量

vim /etc/profile

29 export SW_AGENT_NAME='magedu-python3.8-app1'
30 export SW_AGENT_NAMESPACE='python-project'
31 export SW_AGENT_COLLECTOR_BACKEND_SERVICES='169.254.90.148:11800'  #skaywalking-server地址

source  /etc/profile

vim /apps/django-test/mysite/mysite/settings.py

28 ALLOWED_HOSTS = ["*",]   #修改配置，允许任何主机访问

sw-python -d run python3 manage.py runserver 169.254.90.149:80  #启动服务
```

前端访问169.254.90.149/admin 登录django 后台页面并创建用户
<img width="872" alt="微信图片_20221216142656" src="https://user-images.githubusercontent.com/82318011/208225935-1c0cb081-0c4a-4f4b-b26c-caac768b0b39.png">

到skywalking ui前端验证

<img width="906" alt="微信图片_20221216142711" src="https://user-images.githubusercontent.com/82318011/208225942-fdfe9488-a20b-488e-ab28-b3a402b2fccf.png">


**2.实现OpenResty及后端java服务的全链路请求链路跟踪**

准备编译环境：

```shell
apt  install iproute2  ntpdate  tcpdump telnet traceroute nfs-kernel-server nfs-common  lrzsz tree  openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute  gcc openssh-server lrzsz tree  openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute iotop unzip zip  make

wget https://openresty.org/download/openresty-1.21.4.1.tar.gz  下载openresty-1.21.4.1.tar.gz

tar xvf openresty-1.21.4.1.tar.gz   

cd openresty-1.21.4.1/

./configure --prefix=/apps/openresty  \
--with-luajit \
--with-pcre \
--with-http_iconv_module \
--with-http_realip_module \
--with-http_sub_module \
--with-http_stub_status_module \
--with-stream \
--with-stream_ssl_module    #编译openresty

make && make install

/apps/openresty/bin/openresty  -t   验证配置文件
```

配置skywalking nginx agent  # https://github.com/apache/skywalking-nginx-lua

```shell
mkdir  /data  && cd /data

wget https://github.com/apache/skywalking-nginx-lua/archive/refs/tags/v0.6.0.tar.gz

tar xvf skywalking-nginx-lua-v0.6.0.tar.gz 

cd /apps/openresty/nginx/conf/

vim nginx.conf

include /apps/openresty/nginx/conf/conf.d/*.conf;   #运行openresty时加载conf.d的配置

mkdir  conf.d  

vim  conf.d/www.myserver.com.conf 

 lua_package_path "/data/skywalking-nginx-lua-0.6.0/lib/?.lua;;";

    # Buffer represents the register inform and the queue of the finished segment

    lua_shared_dict tracing_buffer 100m;

    # Init is the timer setter and keeper
    # Setup an infinite loop timer to do register and trace report.
    init_worker_by_lua_block {
        local metadata_buffer = ngx.shared.tracing_buffer
    
        metadata_buffer:set('serviceName', 'myserver-nginx') ---#在skywalking 显示的当前server 名称，用于区分事件是有哪个服务产生的
        -- Instance means the number of Nginx deloyment, does not mean the worker instances
        metadata_buffer:set('serviceInstanceName', 'myserver-nginx-node1') ---#当前示例名称，用户事件是在那台服务器产生的
        -- type 'boolean', mark the entrySpan include host/domain
        metadata_buffer:set('includeHostInEntrySpan', false) ---#在span信息中包含主机信息
    
        -- set randomseed
        require("skywalking.util").set_randomseed()
    
        require("skywalking.client"):startBackendTimer("http://169.254.90.144:12800") #http为12800，java grpc使用11800
    
        -- Any time you want to stop reporting metrics, call `destroyBackendTimer`
        -- require("skywalking.client"):destroyBackendTimer()
    
        -- If there is a bug of this `tablepool` implementation, we can
        -- disable it in this way
        -- require("skywalking.util").disable_tablepool()
    
        skywalking_tracer = require("skywalking.tracer")
    }
    
    server {
        listen 80;
        server_name   www.myserver.com;
        location / {
            root   html;
            index  index.html index.htm;
            #手动配置的一个上游服务名称或DNS名称，在skywalking会显示此名称
            rewrite_by_lua_block {
                ------------------------------------------------------
                -- NOTICE, this should be changed manually
                -- This variable represents the upstream logic address
                -- Please set them as service logic name or DNS name
                --
                -- Currently, we can not have the upstream real network address
                ------------------------------------------------------
                skywalking_tracer:start("www.myserver.com")  #openresty域名，需要加到本地hosts文件
                -- If you want correlation custom data to the downstream service
                -- skywalking_tracer:start("upstream service", {custom = "custom_value"})
            }
            #用于修改响应内容(注入JS)
            body_filter_by_lua_block {
                if ngx.arg[2] then
                    skywalking_tracer:finish()
                end
            } 
            #记录日志
            log_by_lua_block { 
                skywalking_tracer:prepareForReport()
            } 
    
        }
    
        location /myserver {
            default_type text/html;
    
            rewrite_by_lua_block {
                ------------------------------------------------------
                -- NOTICE, this should be changed manually
                -- This variable represents the upstream logic address
                -- Please set them as service logic name or DNS name
                --
                -- Currently, we can not have the upstream real network address
                ------------------------------------------------------
                skywalking_tracer:start("www.myserver.com")
                -- If you want correlation custom data to the downstream service
                -- skywalking_tracer:start("upstream service", {custom = "custom_value"})
            }
    
            proxy_pass http://169.254.90.128;  #后端apache节点  访问地址http://169.254.90.128/myserver
    
            body_filter_by_lua_block {
                if ngx.arg[2] then
                    skywalking_tracer:finish()
                end
            }
    
            log_by_lua_block {
                skywalking_tracer:prepareForReport()
            }
        }
    
        location /hello {
            default_type text/html;
    
            rewrite_by_lua_block {
                ------------------------------------------------------
                -- NOTICE, this should be changed manually
                -- This variable represents the upstream logic address
                -- Please set them as service logic name or DNS name
                --
                -- Currently, we can not have the upstream real network address
                ------------------------------------------------------
                skywalking_tracer:start("www.myserver.com")
                -- If you want correlation custom data to the downstream service
                -- skywalking_tracer:start("upstream service", {custom = "custom_value"})
            }
    
            proxy_pass http://169.254.90.146:8080;  #后端consumer节点，访问地址为http://169.254.90.146:8080/hello?lisi
    
            body_filter_by_lua_block {
                if ngx.arg[2] then
                    skywalking_tracer:finish()
                end
            }
    
            log_by_lua_block {
                skywalking_tracer:prepareForReport()
            }
        }

}
```

/apps/openresty/nginx/sbin/nginx  -t     #检测配置文件语法

/apps/openresty/nginx/sbin/nginx  -s reload  # 重启openresty

登录skywalking web界面逐一验证数据
