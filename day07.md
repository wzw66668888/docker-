**一.了解prometheus各组件的功能，熟悉prometheus的数据采集流程**

prometheus server：主服务，接受外部http请求、收集指标数据、存储指标数据与查询指标数据等

prometheus targets: 静态发现目标后执行指标数据抓取。

service discovery：动态发现目标后执行数据抓取。

push gateway：数据收集代理服务器，收集short-lived jobs任务

prometheus alerting：调用alertmanager组件实现报警通知

data visualization and export： 数据可视化与数据导出

prometheus的数据采集流程：

1.基于静态配置文件或动态发现获取目标

2.向目标URL发起http/https请求

3.目标接受请求并返回指标数据

4.prometheus server接受并数据并对比告警规则，如果触发告警则进一步执行告警动作并存储数据，不触发告警则只进行数据存储

5.grafana进行数据可视化

**二.基于docker或二进制部署prometheus server** 

1.准备docker、及docker-compose环境

```shell
tar xvf docker-20.10.19-binary-install.tar.gz

bash docker-install.sh

git clone https://gitee.com/jiege-gitee/prometheus-docker-compose.git

vim prometheus/prometheus-docker-compose/config/prometheus/prometheus.yml

 30   - job_name: 'prometheus-node'
 31     static_configs:
 32       - targets: ['169.254.90.140:9100','169.254.90.142:9100']  #添加后端监控节点

docker-compose up -d  拉起prometheus-server、node-exporter、grafana镜像
```

到prometheus前端验证
![1](https://user-images.githubusercontent.com/82318011/206614370-8dad7f1d-21ba-41f2-ad65-7d4d1b86f272.png)


**三.基于docker或二进制部署node-exporter，并通过prometheus收集node-exporter指标数据**

###### **1.二进制安装prometheus**

```shell
mkdir -p /apps  &&    cd /apps

wget https://github.com/prometheus/prometheus/releases/download/v2.40.5/prometheus-2.40.5.linux-amd64.tar.gz  #官网下载prometheus安装包

tar xvf prometheus-2.40.5.linux-amd64.tar.gz

ln -sv /apps/prometheus-2.40.5.linux-amd64 /apps/prometheus

vim /etc/systemd/system/prometheus.service  #创建prometheus service启动文件

[Unit]

Description=Prometheus Server

Documentation=https://prometheus.io/docs/introduction/overview/

After=network.target

[Service]

Restart=on-failure

WorkingDirectory=/apps/prometheus/

ExecStart=/apps/prometheus/prometheus --config.file=/apps/prometheus/prometheus.yml --web.enable-lifecycle

[Install]

WantedBy=multi-user.target

systemctl daemon-reload &&  systemctl enable prometheus.service --now
```

登录前端页面访问测试

###### **2.prometheus-node 节点安装node-exporter**

mkdir -p /apps  &&    cd /apps

```shell
 wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz   #官网下载node-exporter 

 tar -xvf node_exporter-1.5.0.linux-amd64.tar.gz 

 ln -sv node_exporter-1.5.0.linux-amd64 node_exporter

vim /etc/systemd/system/node-exporter.service

[Unit]
Description=Prometheus Node Exporter
After=network.target
[Service]
ExecStart=/apps/node_exporter/node_exporter
[Install]
WantedBy=multi-user.target

systemctl daemon-reload &&  systemctl enable node_exporter.service --now
```

到前端验证node_exporter 页面


###### **3.prometheus-server节点添加已安装node_exporter 的节点信息，收集节点指标数据**

```shell
vim  /apps/prometheus/prometheus.yml

 30   - job_name: "prometheus-nodes"
 31     static_configs:
 32       - targets: ["169.254.90.144:9100","169.254.90.147:9100"]

./promtool check config prometheus.yml  #检测配置prometheus配置文件语法

systemctl restart prometheus.service 
```

到prometheus前端验证
![2](https://user-images.githubusercontent.com/82318011/206614878-c322576e-0f3a-4548-acec-2caf55485bda.png)


**四.安装grafana并添加prometheus数据源，导入模板可以图形显示指标数据**

1.安装依赖包

apt-get install -y adduser libfontconfig1

2.官网下载grafana linux 安装包

```shell
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_9.3.1_amd64.deb

dpkg  -i  grafana-enterprise_9.3.1_amd64.deb 

vim /etc/grafana/grafana.ini   #修改配置文件

35 protocol = http

38 http_addr = 0.0.0.0

41 http_port = 3000

systemctl enable grafana-server.service --now
```

登录grafana web 界面，默认账号密码均为admin

点击Configuration-->data source-->add datasource-->prometheus，填写数据源名称和prometheus-server地址保存

导入模板有两种方式用模板id直接导入(要求grafana可以连接公网) Dashboards-->import-->模板id 11074 

第二种官网点击dashboard-Data Source搜索prometheus选择模板下载，Dashboards-->import---->upload json file 导入
![3](https://user-images.githubusercontent.com/82318011/206614958-e3e0c286-2b4c-4b5c-baa7-cf27402fdb79.png)



**五.掌握prometheus的promQL语句的简单使用**

Instant Vector：瞬时向量/瞬时数据,是对目标实例查询到的同一个时间戳的一组时间序列数据(按照时间的推移对数据进存储和展示)，每个时间序列包含单个数据样本，比如node_memory_MemFree_bytes查询的是当前剩余内存(可用内存)就是一个瞬时向量，该表达式的返回值中只会包含该时间序列中的最新的一个样本值，而相应的这样的表达式称之为瞬时向量表达式。

示例：curl 'http://169.254.90.146:9090/api/v1/query' --data 'query=node_memory_MemFree_bytes' --data time=1670466957

Range Vector：范围向量/范围数据,是指在任何一个时间范围内，抓取的所有度量指标数据.比如最近一天的网卡流量趋势图、或最近5分钟的node节点内容可用字节数等。

示例：curl 'http://169.254.90.146:9090/api/v1/query' --data 'query=node_memory_MemFree_bytes{instance="169.254.90.144:9100"}[5m]' --data time=1670468457

scalar：标量/纯量数据,是一个浮点数类型的数据值，使用node_load1获取到时一个瞬时向量后，在使用prometheus的内置函数scalar()将瞬时向量转换为标量。

示例：curl 'http://169.254.90.146:9090/api/v1/query' --data 'query=scalar(sum(node_load1{instance="169.254.90.144:9100"}))' --data time=1670469000

string：简单的字符串类型的数据，目前未使用。

prometheus指标数据类型

Counter:计数器,Counter类型代表一个累积的指标数据，在没有被重启的前提下只增不减(生活中的电表、水表)，比如磁盘I/O总数、Nginx/API的请求总数、网卡流经的报文总数等。

Gauge:仪表盘,Gauge类型代表一个可以任意变化的指标数据，值可以随时增高或减少，如带宽速率、CPU负载、内存利用率、nginx 活动连接数等。

Histogram：累积直方图，Histogram会在一段时间范围内对数据进行采样(通常是请求持续时间或响应大小等),假如每分钟产生一个当前的活跃连接数，那么一天24小时*60分钟=1440分钟就会产生1440个数据，查看数据的每间隔的绘图跨度为2小时，那么2点的柱状图(bucket)会包含0点到2点即两个小时的数据，而4点的柱状图(bucket)则会包含0点到4点的数据，而6点的柱状图(bucket)则会包含0点到6点的数据，可用于统计从当天零点开始到当前时间的数据统计结果，如http请求成功率、丢包率等，比如ELK的当天访问IP统计。

Summary：摘要图，也是一组数据，默认统计选中的指标的最近10分钟内的数据的分位数，可以指定数据统计时间范围，基于分位数(Quantile),亦称分

位点,是指用分割点(cut point)将随机数据统计并划分为几个具有相同概率的连续区间，常见的为四分位，四分位数是将数据样本统计后分成四个区间，将范围内的数据进行百分比的占比统计,从0到1，表示是0%~100%，(0%~25%,%25~50%,50%~75%,75%~100%),利用四分位数，可以快速了解数据的大概统计结果。

node_memory_MemTotal_bytes #查询node节点总内存大小

node_memory_MemFree_bytes #查询node节点剩余可用内存

node_memory_MemTotal_bytes{instance="169.254.90.144:9100"} #基于标签查询指定节点的总内存

node_memory_MemFree_bytes{instance="169.254.90.144:9100"} #基于标签查询指定节点的可用内存

node_disk_io_time_seconds_total{device="sda"} #查询指定磁盘的每秒磁盘io

node_filesystem_free_bytes{device="/dev/sda1" ,fstype="xfs" ,mountpoint="/"} #查看指定磁盘的磁盘剩余空间

promQL标签匹配：

= :选择与提供的字符串完全相同的标签，精确匹配

!= :选择与提供的字符串不相同的标签，去反

=~ :选择正则表达式与提供的字符串（或子字符串）相匹配的标签

!~ :选择正则表达式与提供的字符串（或子字符串）不匹配的标签

示例：

node_filesystem_free_bytes{instance="169.254.90.147:9100"}    #精确匹配

node_filesystem_free_bytes{instance!="169.254.90.144:9100"}    \#取反

node_filesystem_free_bytes{instance=~"169.254.90.14.*:9100"}  #包含正则且匹配

node_filesystem_free_bytes{instance!~"169.254.90.13.*:9100"}  #包含正则且取反

promQL时间范围：

s - 秒  m - 分钟  h - 小时  d - 天  w - 周  y - 年

node_memory_MemTotal_bytes{instance="172.31.1.181:9100"}[5m]  \#区间向量表达式，选择以当前时间为基准，查询指定节点node_memory_MemTotal_bytes指标5分钟内的数据

promQL运算符：

\+ 加法   - 减法  *  乘法   / 除法   % 模   ^ 幂(N次方)

node_memory_MemFree_bytes/1024/1024    #将内存进行单位从字节转行为兆

node_disk_read_bytes_total{device="sda"} + node_disk_written_bytes_total{device="sda"}  \#计算磁盘读写数据量

promQL聚合运算：

max() #最大值   min() #最小值   avg() #平均值

max(node_network_receive_bytes_total)by(instance)  #计算每个节点的最大的流量值

max(rate(node_network_receive_bytes_total[5m]))by(instance) #计算每个节点最近五分钟每个device的最大流量

max(prometheus_http_requests_total)   \#最近总共请求数为xxxx次，用于计算返回值的总数(如http请求次数)

count(prometheus_http_requests_total)  #一共n条返回的数据，可以用于统计节点数、pod数量等

count_values("node_version" ,node_os_version)  #统计不同的系统版本节点有多少

abs(sum(prometheus_http_requests_total{handler="/metrics"}))    #返回指标数据的值

absent(sum(prometheus_http_requests_total{handler="/metrics"}))  \#如果监指标有数据就返回空，如果监控项没有数据就返回1，可用于对监控项设置告警通知(如果返回值等于1就触发告警通知)

stddev(prometheus_http_requests_total)  #标准差，5+5=10,1+9=10,1+9这一组的数据差异就大，在系统表示数据波动较大，不稳定

stdvar(prometheus_http_requests_total)  #求方差

topk(3,prometheus_http_requests_total)  #取http请求数从大到小的前3个

bottomk(3, prometheus_http_requests_total)  #取http请求从小到大的前3个

rate() #rate函数是专门搭配counter数据类型使用函数，rate会取指定时间范围内所有数据点，算出一组速率，然后取平均值作为结果,适合用于计算数据相对平稳的数据。

rate(prometheus_http_requests_total[5m])

rate(apiserver_request_total{code=~ "^(?:2..)$"}[5m])

rate(node_network_receive_bytes_total[5m])

irate() #函数也是专门搭配counter数据类型使用函数，irate取的是在指定时间范围内的最近两个数据点来算速率，适合计算数据变化比较大的数据，显示的数据相对比较准确,因此irate适合快速变化的计数器（counter），而rate适合缓慢变化的计数器（counter）

irate(prometheus_http_requests_total[5m])

irate(node_network_receive_bytes_total[5m])

irate(apiserver_request_total{code=~ "^(?:2..)$"}[5m])

\#by，在计算结果中，只保留by指定的标签的值，并移除其它所有的

sum(rate(node_network_receive_packets_total{instance=~ ".*"}[10m])) by (instance,device)  

sum(rate(node_memory_MemFree_bytes[5m])) by (increase)    #只保留5分钟增长的

sum(prometheus_http_requests_total) without (instance,job)  \#without，从计算结果中移除列举的instance,job标签，保留其它标签



**六.部署prometheus联邦集群并实现指标数据收集**

环境准备：一台prometheus-server节点,两台prometheus-server 联邦节点，三台被监控节点

###### 1.两台prometheus-server 联邦节点部署prometheus

```shell
mkdir -p /apps  &&    cd /apps

wget https://github.com/prometheus/prometheus/releases/download/v2.40.5/prometheus-2.40.5.linux-amd64.tar.gz  #官网下载prometheus安装包

tar xvf prometheus-2.40.5.linux-amd64.tar.gz

ln -sv /apps/prometheus-2.40.5.linux-amd64 /apps/prometheus

vim /etc/systemd/system/prometheus.service  #创建prometheus service启动文件

[Unit]

Description=Prometheus Server

Documentation=https://prometheus.io/docs/introduction/overview/

After=network.target

[Service]

Restart=on-failure

WorkingDirectory=/apps/prometheus/

ExecStart=/apps/prometheus/prometheus --config.file=/apps/prometheus/prometheus.yml --web.enable-lifecycle

[Install]

WantedBy=multi-user.target

systemctl daemon-reload &&  systemctl enable prometheus.service --now
```

###### 2.被监控节点部署node_exporter

 wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz   #官网下载node-exporter 

```shell
mkdir -p /apps  &&    cd /apps

 tar -xvf node_exporter-1.5.0.linux-amd64.tar.gz 

 ln -sv node_exporter-1.5.0.linux-amd64 node_exporter

vim /etc/systemd/system/node-exporter.service

[Unit]
Description=Prometheus Node Exporter
After=network.target
[Service]
ExecStart=/apps/node_exporter/node_exporter
[Install]
WantedBy=multi-user.target

systemctl daemon-reload &&  systemctl enable node_exporter.service --now
```

###### 3.两台prometheus 联邦节点修改配置文件添加job到被监控节点采集数据

```shell
vim  /apps/prometheus/prometheus.yml

 31   - job_name: "prometheus-idc1"
 32     static_configs:
 33       - targets: ["169.254.90.134:9100"]
 
 systemctl restart prometheus.service

vim  /apps/prometheus/prometheus.yml

 30   - job_name: "prometheus-idc2"
 31     static_configs:
 32       - targets: ["169.254.90.135:9100","169.254.90.137:9100"]
 
 systemctl restart prometheus.service
```

前端验证prometheus targets状态

![4](https://user-images.githubusercontent.com/82318011/206615065-4d24a042-a037-4166-9af7-0d0a45589a40.png)

![5](https://user-images.githubusercontent.com/82318011/206615077-3f84df22-52b9-47fe-be3e-a48c83f7444f.png)



###### 4.配置prometheus-server通过联邦节点收集被监控节点指标数据

```shell
vim  /apps/prometheus/prometheus.yml

  - job_name: 'prometheus-federate-2.102'
    scrape_interval: 10s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
      - '{job="prometheus"}'
      - '{__name__=~"job:.*"}'
      - '{__name__=~"node.*"}'
        static_configs:
    - targets:
      - '169.254.90.145:9090'
  - job_name: 'prometheus-federate-2.103'
    scrape_interval: 10s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
      - '{job="prometheus"}'
      - '{__name__=~"job:.*"}'
      - '{__name__=~"node.*"}'
        static_configs:
    - targets:
      - '169.254.90.147:9090'
      
curl -X POST http://172.31.1.101:9090/-/reload  #不重启刷新prometheus配置
```

5.登录prometheus web页面验证通过联邦节点收集的node-exporter指标数据
![6](https://user-images.githubusercontent.com/82318011/206615124-b4e40fe4-ab1d-442b-8e9c-722aa6055440.png)


6.登录grafana页面查看监控页面
![7](https://user-images.githubusercontent.com/82318011/206615166-c7b6db9b-b4da-4bb3-9518-f2820842381a.png)



**扩展**
**基于pushgateway实现指标数据收集**

1.pushgateway用于临时的指标数据收集

2.pushgateway不支持数据拉取(pull模式)，需要客户端主动将数据推送给pushgateway

3.pushgateway可以单独运行在一个节点，然后需要自定义监控脚本把需要监控的主动推送给pushgateway的API接口，然后pushgateway再等待prometheus server抓取数据，即pushgateway本身没有任何抓取监控数据的功能，目前pushgateway只能被动的等待数据从客户端进行推送。

--persistence.file="" #数据保存的文件，默认只保存在内存中   --persistence.interval=5m #数据持久化的间隔时间

###### 部署pushgateway

```shell
wget https://github.com/prometheus/pushgateway/releases/download/v1.5.1/pushgateway-1.5.1.linux-amd64.tar.gz

tar -xvf pushgateway-1.5.1.linux-amd64.tar.gz 

ln -sv pushgateway-1.5.1.linux-amd64 pushgateway

vim /etc/systemd/system/pushgateway.service

[Unit]
Description=Prometheus pushgateway
After=network.target
[Service]
ExecStart=/apps/pushgateway/pushgateway
[Install]
WantedBy=multi-user.target

systemctl daemon-reload     && systemctl enable pushgateway  --now
```

前端验证pushgateway

pushgateway监听在90991端口，并且可以通过http://169.254.90.145:9091/metrics对外提供指标数据抓取接口

客户端推送单条指标数据要到 PushGateway 中，可以通过其提供的 API 标准接口来添加，默认 URL 地址为：

http://<ip>:9091/metrics/job/<JOBNAME>{/<LABEL_NAME>/<LABEL_VALUE>}

其中<JOBNAME>是必填项，是job的名称，后边可以跟任意数量的标签对，一般我们会添加一个instance/<INSTANCE_NAME>实例名称标签，来方便区分各个指标是在哪个节点产生的。

示例：推送一个job名称为mytest_job，key为mytest_metric值为2022   

echo "mytest_metric 2088" | curl --data-binary @- http://169.254.90.145:9091/metrics/job/mytest_job  

到浏览器http://169.254.90.145:9091 验证

prometheus配置文件添加pushgateway地址进行数据采集

```shell
vim /apps/prometheus/prometheus.yml 

 34   - job_name: "prometheus-pushgateway"
 35     static_configs:
 36       - targets: ["169.254.90.145:9091"]

systemctl restart prometheus.service  #重启prometheus服务
```

到前端验证prometheus是否采集到pushgateway的数据

客户端推送数据到pushgateway有两种方式：

方式一 ：

```shell
cat <<EOF | curl --data-binary @- http://172.31.1.102:9091/metrics/job/test_job/instance/172.31.2.181

\#TYPE node_memory_usage gauge

node_memory_usage 4311744512

\# TYPE memory_total gauge

node_memory_total 103481868288

EOF
```

方式二：

节点基于自定义脚本实现数据的收集和推送：

```shell
vim memory_monitor.sh

\#!/bin/bash

total_memory=$(free |awk '/Mem/{print $2}')

used_memory=$(free |awk '/Mem/{print $3}')

job_name="custom_memory_monitor"

instance_name=`ifconfig eth0 | grep -w inet | awk '{print $2}'` 

pushgateway_server="http://172.31.1.102:9091/metrics/job"

cat <<EOF | curl --data-binary @- ${pushgateway_server}/${job_name}/instance/${instance_name}

\#TYPE custom_memory_total gauge

custom_memory_total $total_memory

\#TYPE custom_memory_used gauge

custom_memory_used $used_memory

EOF

bash memory_monitor.sh
```

登录pushgateway验证
![8](https://user-images.githubusercontent.com/82318011/206615235-7dc61649-adbc-4991-b00f-832337217fc1.png)


登录prometheus验证
![9](https://user-images.githubusercontent.com/82318011/206615271-d0274038-2338-41c0-b421-4ae8e232aff5.png)

![10](https://user-images.githubusercontent.com/82318011/206615721-e7dea846-edbe-4e82-af70-aed21910ee78.png)


