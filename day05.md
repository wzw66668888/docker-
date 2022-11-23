##### 一. 完全基于pipline实现完整的代码部署流水线

```shell
mkdir -p /opt/app1/build  &&  vim  /opt/app1/build/Jenkinsfile 

\`#!groovy`

`pipeline {`

  `agent any //全局必须带有agent,表明此pipeline执行节点`

  `//agent { label 'jenkins node1' }`

  `options {`

​    `buildDiscarder(logRotator(numToKeepStr: '5')) //保留最近5个构建历史`

​    `disableConcurrentBuilds()  //禁用并发构建`

  `}`

  `//声明环境变量`

  `environment{`

​    `//定义镜像仓库地址`

​    `def GIT_URL = "ssh://git@169.254.90.132:36000/magedu/app1.git"`

​    `//镜像仓库变量`

​    `def HARBOR_URL = 'harbor.magedu.net'`

​    `//镜像项目变量`

​    `def IMAGE_PROJECT = 'myserver'`

​    `//镜像名称变量`

​    `IMAGE_NAME = 'nginx'`

​    `def DATE = sh(script:"date +%F_%H-%M-%S", returnStdout: true).trim()  //基于shell命令获取当前时间`

``   

  `}`



   `//参数定义`

  `parameters {`

​    `string(name: 'BRANCH', defaultValue: 'develop', description: 'branch select') //字符串参数，会配置在jenkins的参数化构建过程中`

​    `choice(name: 'DEPLOY_ENV', choices: ['develop', 'production'], description: 'deploy env') //选项参数，会配置在jenkins的参数化构建过程中`

  `}`



  `stages{`

​    `stage("code clone"){`

​      `//#agent { label 'master' }  //具体执行的步骤节点，非必须`

​      `steps{`

​        `deleteDir() //删除workDir 当前目录`

​        `script{`

​          `if ( env.BRANCH == 'main' ){`

​            `git branch: 'main', credentialsId: 'f022d876-dd08-4f4a-8025-315c8c91deb6', url: 'ssh://git@169.254.90.132:36000/magedu/app1.git'`

​          `}else if ( env.BRANCH == 'develop' ){`

​            `git branch: 'develop', credentialsId: 'f022d876-dd08-4f4a-8025-315c8c91deb6', url: 'ssh://git@169.254.90.132:36000/magedu/app1.git'`

​          `}else {`

​            `echo '您传递的分支参数BRANCH ERROR，请检查分支参数是否正确'`

​          `}`

​          `GIT_COMMIT_TAG = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim() //获取clone完成的分支tagId 用于镜像做tag`

​          `}`

​        `}`

​      `}`

​    `stage("sonarqube-scanner"){`

​      `// #agent { label 'master'} //具体执行步骤的节点，非必须`

​      `steps{`

​        `dir ('$env.WORKSPACE'){`

​          `// some block`

​          `sh '/apps/sonar-scanner/bin/sonar-scanner -Dsonar.projectKey=magedu -Dsonar.projectName=magedu-app1 -Dsonar.projectVersion=1.0  -Dsonar.sources=./ -Dsonar.language=py -Dsonar.sourceEncoding=UTF-8'`

​          `}`

``        

​      `}`

​    `}`

​    `stage("code build"){`

​      `//#agent { label 'master' }`

​      `steps{`

​          `// some block`

​          `sh 'tar czvf frontend.tar.gz ./index.html ./images'`

``        

​      `}`

​    `}`

​    `stage("file sync"){// SSH Pipeline Steps`

​      `// agent { label 'master'}`

​      `steps{`

​        `script {`

​          `stage('file copy') {`

​            `def remote = [:]`

​            `remote.name = 'test'`

​            `remote.host = '169.254.90.128'`

​            `remote.port = 36000`

​            `remote.user = 'root'`

​            `remote.password = '123456'`

​            `remote.allowAnyHosts = true`

​            `//sshCommand remote: remote, command: "docker images"  // 执行远程命令`

​              `//sshCommand remote: remote, command: "docker ps"`

​              `sshPut remote: remote, from: 'frontend.tar.gz', into: '/opt/ubuntu-dockerfile/'  //将本地文件put到远端主机`

​          `}`

``         

​        `}`

​      `}`

​    `}`

​    `stage('image build'){//SSH Pipeline Steps`

​      `//#agent { label 'master' }`

​      `steps{`

​        `script {`

​          `stage('image put'){`

​            `def remote = [:]`

​            `remote.name = 'test'`

​            `remote.host = '169.254.90.128'`

​            `remote.port = 36000`

​            `remote.user = 'root'`

​            `remote.password = '123456'`

​            `remote.allowAnyHosts = true`

​              `sshCommand remote: remote, command: "cd /opt/ubuntu-dockerfile && bash build-command.sh ${GIT_COMMIT_TAG}-${DATE}"`

​          `}`

``        

​        `}`

​      `}`

​    `}`

​    `stage('docker-compose image update'){`

​      `steps{`

​        `sh """`

​          `ssh -p 36000 root@169.254.90.128 "echo ${DATE} && cd /data/magedu-app1 && sed -i 's#image: harbor.magedu.net/myserver/nginx:.*#image: harbor.magedu.net/myserver/nginx:${GIT_COMMIT_TAG}-${DATE}#' docker-compose.yml"`

​        `"""`

​      `}`

​    `}`

​    `stage('docker-compose app update'){`

​      `steps{`

​        `//sh """`

​        `//   ssh root@172.31.6.202 "echo ${DATE} && cd /data/magedu-app1 && docker-compose pull && docker-compose up -d"`

​        `//"""`

​        `//}`

​        `script {`

​          `stage('image update'){`

​            `def remote = [:]`

​            `remote.name = 'docker-server'`

​            `remote.host = '169.254.90.128'`

​            `remote.port = 36000`

​            `remote.user = 'root'`

​            `remote.password = '123456'`

​            `remote.allowAnyHosts = true`

​              `sshCommand remote: remote, command: "cd /data/magedu-app1 && docker-compose pull && docker-compose up -d"`

​          `}`

​        `}`

​      `}`

​    `}`

​    `stage('send email') {`

​      `steps {`

​       `sh 'echo send email'`

​      `}`

​      `post {`

​      `always {`

​       `script {`

​        `mail to: '2968687730@qq.com',`

​          `subject: "Pipeline Name: ${currentBuild.fullDisplayName}",`

​          `body: " ${env.JOB_NAME} -Build Number-${env.BUILD_NUMBER} \n Build URL-'${env.BUILD_URL}' "`

​        `}`

​         `}`



​        `}`

​      `}`

​    `}`

  `}`

git add . && git commit -m "add  jenkinsfile" && git pull 
```

登录jenkens web页面创建任务配置里将Pipeline script 改为Pipeline script from  scm，点击立即构建
![image-20221116200331999](https://user-images.githubusercontent.com/82318011/203564297-aefe343f-ade9-4e50-89ec-a80f53d9f914.png)


![image-20221122124524288](https://user-images.githubusercontent.com/82318011/203564346-c8992dae-11cb-4487-98ba-5caee2d8f274.png)



##### 二. 熟悉ELK各组件的功能、elasticsearch的节点角色类型

ELK由elasticsearch、logstash、kibana组成，elasticsearch负责数据存储及检索，logstash负责日志收集、日志处理并发送至elastersearch,kibana负责从ES中读取数据进行可视化展示及数据管理。

elastersearch主要节点类型：

data node:  数据节点，负责数据的存储，如分片的创建及删除、数据的读写、数据的更新、数据的删除等操作。

master节点： 主节点，负责index的创建、删除，分片的分配，node节点的添加、删除，node节点宕机时将状态通告至其它可用node节点，一个ES集群只有一个活跃的master node节点，其它非活跃的master备用节点将等待master宕机以后进行新的master的竞选。

client node/coordinating-node：客户端节点或协调节点，负责将数据读写请求转发data node、将集群管理相关的操作转发到master node，客户端节点只作为集群的访问入口、其不存储任何数据，也不参与master角色的选举。

Ingest节点：预处理节点，在检索数据之前可以先对数据做预处理操作(Ingest pipelines,数据提取管道)，可以在管道对数据实现对数据的字段删除、文本提取等操作。所有节点其实默认都是支持 Ingest 操作的，也可以专门将某个节点配置为 Ingest 节点。



##### 三.熟悉索引、doc、分片与副本的概念

index(索引)：es中一类相同类型的数据(doc),在逻辑上通过同一个index进行查询、修改与删除等操作

document(文档): es的文档简称doc，存储在elasticsearch的json数据

shard(分片)：shard是对index的逻辑拆分存储，分片可以是一个也可以是多个，多个分片合并起来就是index索引的所有数据。

replica(副本)： replica是一个分片跨主机的完整备份，分为主分片和副本分片，数据写入主分片时立即同步到副本分片，以实现数据高可用及主分片宕机的故障转移，副本分片可以读，多副本分片可以提高ES集群的读性能，replica只有在主分片宕机以后才会给提升为主分片继续写入数据，并为其添加新的副本分片。



##### 四.掌握不同环境的ELK部署规划，基于deb或二进制部署elasticsearch集群

二进制部署es集群

1. **环境准备**：一台2C,4G, 2台2C2G的虚拟机或云主机，官网下载elasticsearch-8.5.1-linux-x86_64.tar.gz

   内核参数优化:(三台都要做)

   ```shell
   echo "vm.max_map_count=262144" >> /etc/sysctl.conf
   ```

   **主机名解析：**

   ```shell
   vim /etc/hosts
   
   169.254.90.140  es-node
   169.254.90.142  es-node1
   169.254.90.137  es-node2
   ```

   **资源limits优化：**

   ```shell
   vim /etc/security/limits.conf
   
   # End of file
   
   root soft core unlimited
   
   root hard core unlimited
   
   root soft nproc 1000000
   
   root hard nproc 1000000
   
   root soft nofile 1000000
   
   root hard nofile 1000000
   
   root soft memlock 32000
   
   root hard memlock 32000
   
   root soft msgqueue 8192000
   
   root hard msgqueue 8192000
   
   * soft core unlimited
   
   * hard core unlimited
   
   * soft nproc 1000000
   
   * hard nproc 1000000
   
   * soft nofile 1000000
   
   * hard nofile 1000000
   
   * soft memlock 32000
   ```

​        **创建普通用户运行环境：**

```shell
  groupadd -g 2888 elasticsearch  && useradd -u 2888 -g 2888 -r -m -s /bin/bash  elasticsearch

  echo "elasticsearch:123456" | chpasswd  #设置用户密码

  mkdir /data/esdata /data/eslogs /apps  -p
```

​         **部署elasticsearch集群：**

```shell
  tar xvf elasticsearch-8.5.1-linux-x86_64.tar.gz

  ln -sv /apps/elasticsearch-8.5.1 /apps/elasticsearch

  chown elasticsearch.elasticsearch /data /apps/ -R

  reboot
```

​        **xpack认证签发环境：**

​         签发证书:

```she
    su - elasticsearch
```

```
    cd /apps/elasticsearch
```

​    vim instances.yml

  - ```yaml
    instances:
      - name: "es-node"
        ip: 
          - "169.254.90.140"
      - name: "es-node1"
        ip: 
          - "169.254.90.142"
      - name: "es-node2"
        ip: 
          - "169.254.90.137"
    ```
    
    ```shell
    #⽣成CA私钥，默认名字为elastic-stack-ca.p12
    
    /apps/elasticsearch$ bin/elasticsearch-certutil ca
    
    #⽣产CA公钥，默认名称为elastic-certificates.p12
    
    /apps/elasticsearch$ bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
    
    #签发elasticsearch集群主机证书：
    
    elasticsearch@es1:/apps/elasticsearch$ bin/elasticsearch-certutil cert --silent --in instances.yml --out certs.zip --pass magedu123 --ca elastic-stack-ca.p12 #指定证书密码为
    
    magedu123
    
    Enter password for CA (elastic-stack-ca.p12) : #CA私钥如果没有密码就直接按回⻋确认
    
    证书分发：
    
    #本机(node1)证书：
    
    elasticsearch@es1:/apps/elasticsearch$ unzip certs.zip
    
    elasticsearch@es1:/apps/elasticsearch$ mkdir config/certs
    
    elasticsearch@es1:/apps/elasticsearch$ cp -r  es-node/es-node.p12  config/certs/
    
    node2证书：
    
    elasticsearch@es2:/apps/elasticsearch$ mkdir config/certs
    
    elasticsearch@es1:/apps/elasticsearch$ scp -r  -P 36000  es-node1/  elasticsearch@es-node1:/apps/elasticsearch/config/certs
    
    node3证书：
    
    elasticsearch@es3:/apps/elasticsearch$ mkdir config/certs
    
    elasticsearch@es1:/apps/elasticsearch$ scp -r  -P 36000  es-node2/ elasticsearch@es-node2:/apps/elasticsearch/config/certs
    
    
    
    #⽣成 keystore ⽂件(keystore是保存了证书密码的认证⽂件magedu123)
    
    elasticsearch@es1:/apps/elasticsearch$ ./bin/elasticsearch-keystore create #创建keystore⽂件
    
    Created elasticsearch keystore in /apps/elasticsearch/config/elasticsearch.keystore
    
    elasticsearch@es1:/apps/elasticsearch$ ./bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
    
    Enter value for xpack.security.transport.ssl.keystore.secure_password: #magedu123
    
    elasticsearch@es1:/apps/elasticsearch$ ./bin/elasticsearch-keystore add
    
    xpack.security.transport.ssl.truststore.secure_password
    
    Enter value for xpack.security.transport.ssl.truststore.secure_password: #magedu123
    ```
    
    
    
    ```
    分发认证文件：
    
    node2:
    
    elasticsearch@es1:/apps/elasticsearch$ scp -P 36000  es-node1/es-node1.p12
    
    elasticsearch@es-node1:/apps/elasticsearch/config/certs
    
    node3:
    
    elasticsearch@es1:/apps/elasticsearch$ scp -P 36000  es-node2/es-node2.p12 
    
    elasticsearch@es-node2:/apps/elasticsearch/config/certs
    ```
    
    
    
    **编辑es配置文件**：
    
    ```shell
    vim config/elasticsearch.yml
    
    es-node1:
    
    17  cluster.name: magedu-es-cluster1
    
    23  node.name: node-1
    
    33  path.data: /data/esdata
    
    37  path.logs: /data/eslogs
    
    56  network.host: 0.0.0.0
    
    61  http.port: 9200
    
    70  discovery.seed_hosts: ["169.254.90.140", "169.254.90.142", "169.254.90.137"]
    
    74   cluster.initial_master_nodes: ["169.254.90.140", "169.254.90.142", "169.254.90.137"]
    
    88  action.destructive_requires_name: true
    
    90 xpack.security.enabled: true
    91 xpack.security.transport.ssl.enabled: true
    92 xpack.security.transport.ssl.keystore.path: /apps/elasticsearch/config/certs/es-node.p12
    93 xpack.security.transport.ssl.truststore.path: /apps/elasticsearch/config/certs/es-node.p12
    ```
    
    es-node2和node3只需要修改node-name分别为node-2、node-3,其余配置一致
    
    **各node节点配置service⽂件:**
    
    ```shell
    # vim /lib/systemd/system/elasticsearch.service
    
    [Unit]
    
    Description=Elasticsearch
    
    Documentation=http://www.elastic.co
    
    Wants=network-online.target
    
    After=network-online.target
    
    [Service]
    
    RuntimeDirectory=elasticsearch
    
    Environment=ES_HOME=/apps/elasticsearch
    
    Environment=ES_PATH_CONF=/apps/elasticsearch/config
    
    Environment=PID_DIR=/apps/elasticsearch
    
    WorkingDirectory=/apps/elasticsearch
    
    User=elasticsearch
    
    Group=elasticsearch
    
    ExecStart=/apps/elasticsearch/bin/elasticsearch --quiet
    
    # StandardOutput is configured to redirect to journalctl since
    
    # some error messages may be logged in standard output before
    
    # elasticsearch logging system is initialized. Elasticsearch
    
    # stores its logs in /var/log/elasticsearch and does not use
    
    # journalctl by default. If you also want to enable journalctl
    
    # logging, you can simply remove the "quiet" option from ExecStart.
    
    StandardOutput=journal
    
    StandardError=inherit
    
    # Specifies the maximum file descriptor number that can be opened by this process
    
    LimitNOFILE=65536
    
    # Specifies the maximum number of processes
    
    LimitNPROC=4096
    
    # Specifies the maximum size of virtual memory
    
    LimitAS=infinity
    
    # Specifies the maximum file size
    
    LimitFSIZE=infinity
    
    # Disable timeout logic and wait until process is stopped
    
    TimeoutStopSec=0
    
    # SIGTERM signal is used to stop the Java process
    
    KillSignal=SIGTERM
    
    # Send the signal only to the JVM rather than its control group
    
    KillMode=process
    
    # Java process is never killed
    
    SendSIGKILL=no
    
    # When a JVM receives a SIGTERM signal it exits with code 143
    
    SuccessExitStatus=143
    
    [Install]
    
    WantedBy=multi-user.target
    ```
    
    **各节点同时启动elasticsearch**：
    
    systemctl daemon-reload && systemctl start elasticsearch.service &&systemctl enable elasticsearch.service
    
    tail -f  /data/eslogs/magedu-es-cluster1.log  无报错   ps -ef |grep elastic可以看到elastic 进程则es集群部署完成
    
    

##### 五.了解elasticsearch API的简单使用，安装head插件管理ES的数据

```shell
curl -u magedu:123456 -X GET http://169.254.90.140:9200   \#获取集群状态

curl -u magedu:123456 -X GET http://169.254.90.140:9200/_cat  #查看集群支持的操作

curl -u magedu:123456 -X GET http://169.254.90.140:9200/_cat/master?v  \#获取master信息

curl -u magedu:123456 -X GET http://169.254.90.140:9200/_cat/nodes?v  \#获取node节点信息

curl -u magedu:123456 -X GET http://169.254.90.140:9200/_cat/health?v  #获取集群心跳信息

curl -u magedu:123456 -X PUT http://169.254.90.140:9200/test_index?pretty  \#创建索引test_index，pretty 为格式序列化

curl -u magedu:123456 -X GET http://169.254.90.140:9200/test_index?pretty  \#查看test_index索引

curl -u magedu:123456 -X POST "http://169.254.90.140:9200/index_test/_doc/1?pretty" -H 'Content-Type:application/json' -d '{"name":"jack","age":19}'   \#上传数据

curl -u magedu:123456 -X GET "http://169.254.90.140:9200/index_test/_doc/1?pretty"  #查看上传的数据

curl -u magedu:123456 -X GET "http://169.254.90.140:9200/index_test/_settings?pretty" \#查看索引设置

curl -u magedu:123456 -X PUT http://169.254.90.140:9200/index_test1/_settings -H 'Content-Type:application/json' -d '{"number_of_replicas": 2}'  \#修改副本数,副本数可动态调整

curl -u magedu:123456 -X POST "http://169.254.90.140:9200/index_test/_close" #关闭索引

curl -u magedu:123456 -X POST "http://169.254.90.140:9200/index_test/_open?pretty" #打开索引

 curl -u magedu:123456 -X DELETE "http://169.254.90.140:9200/test_index?pretty"  #删除索引

curl -u magedu:123456 -X PUT http://169.254.90.140:9200/_cluster/settings -H 'Content-Type: application/json' 

-d' {"persistent" :

 {"cluster.max_shards_per_node" : "1000000"}

}'  #修改集群每个节点的最大可分配的分片数，es7默认为1000，用完后创建新的分片报错误状态码400

curl -u magedu:123456 -X PUT http://169.254.90.140:9200/_cluster/settings -H 'Content-Type: application/json' -d'
{
"persistent": {
"cluster.routing.allocation.disk.watermark.low": "95%"
,
"cluster.routing.allocation.disk.watermark.high": "95%"
}
}'   #设置磁盘最低和最高使用百分比95%，默认85%不会在当前节点创新新的分配副本、90%开始将副本移动至其它节点、95所有索引只读。
```

安装head插件

在谷歌浏览器中设置———>扩展程序———>谷歌商店搜索elastic head———>点击添加至chrome

![image-20221123144424473](https://user-images.githubusercontent.com/82318011/203564995-ba533391-6bf9-4798-bb93-7c4c03fa40bd.png)

![image-20221123144514364](https://user-images.githubusercontent.com/82318011/203565063-c6a0285b-75f5-4937-a2fe-fa5b770ab6ef.png)

![image-20221123144611848](https://user-images.githubusercontent.com/82318011/203565101-454970bf-3c1b-499f-9d25-6af41b898747.png)


##### 六.安装logstash收集不同类型的系统日志并写入到ES 的不同index

准备环境: 一台2C2G的虚拟机和云主机，官网下载logstash-8.5.1-amd64.deb 上传到主机

dpkg -i logstash-8.5.1-amd64.deb  #安装deb包

```shell
cd  /etc/logstash/conf.d   vim log-file.conf 
input{
  file {
    path => "/var/log/syslog"
    stat_interval => "1"
    start_position => "beginning"
    type => "syslog"
  }
  file {
    path => "/var/log/auth.log"
    stat_interval => "1"
    start_position => "beginning"
    type => "authlog"
}
}


output {
  if [type] == "syslog"{
  elasticsearch {
    hosts => ["169.254.90.140:9200"]
    index => "magedu-app1-syslog-%{+yyyy.MM.dd}"
    user => "magedu"
    password => "123456"
  }
}
  if [type] == "authlog"{
  elasticsearch {
    hosts => ["169.254.90.140:9200"]
    index => "magedu-app1-authlog-%{+yyyy.MM.dd}"
    user => "magedu"
    password => "123456"
  }
}
}

systemctl restart logstash.service
```




##### 七.安装kibana、查看ES集群的数据

上传kibana-8.5.1-amd64.deb 包到es-node1节点

dpkg -i kibana-8.5.1-amd64.deb

vim /etc/kibana/kibana.yml

  6  server.port: 5601

 11 server.host: "0.0.0.0"

 43  elasticsearch.hosts: ["http://169.254.90.142:9200"]

 49  elasticsearch.username: "kibana_system"

 50  elasticsearch.password: "123456"

144  i18n.locale: "zh-CN"

systemctl enable kibana.service  --now

lsof -i:5601

tail -f /var/log/kibana/kibana.log

登录kibana web页面 用户密码是elasticsearch设置的密码

在前端找到Stack Management——>数据视图——>创建数据视图

![image-20221123190257643](https://user-images.githubusercontent.com/82318011/203565395-f8aba470-a094-44fc-b575-3b311eb556c0.png)

前端点击discovery就可以看到es集群数据



#####  扩展:了解heartbeat和metricbeat的使用

heartbeat: Heartbeat 能够通过 ICMP、TCP和 HTTP 进行 ping 检测主机可用性、检测网站可用性

将heartbeat-8.5.1-amd64.deb包上传到服务器上

```shell
dpkg -i heartbeat-8.5.1-amd64.deb

vim  /etc/heartbeat/heartbeat.yml

 26   enabled: true  #false改为true

 28   id: magedu-http-monitor-id

 30   name: magedu-http-monitor-name

 32   urls: ["www.baidu.com","www.magedu.com"]

 34   schedule: '@every 10s'

 36   timeout: 5s

 43 - type: icmp
 44   enabled: true
 45   id: icmp-monitor
 46   name: icmp-ip-monitor
 47   schedule: '*/5 * * * * * *'
 48   hosts: ["169.254.90.140","169.254.90.2"]

 50 - type: tcp
 51   enabled: true
 52   id: myhost-tcp-echo
 53   name: My Host TCP Echo
 54   hosts: ["169.254.90.137:9200","169.254.90.142:9200","169.254.90.140:92    00"]  # default TCP Echo Protocol
 55   #check.send: "Check"
 56   #check.receive: "Check"
 57   schedule: '@every 5s'

 92   host: "169.254.90.140:5601"
 93   setup_kibana_username: "magedu"
 94   setup_kibana_passwd: "123456"

 121   hosts: ["169.254.90.142:9200"]

 128   username: "elastic"
 129   password: "123456"

  systemctl enable heartbeat-elastic.service --now
```

kibana验证数据：

Observability-->监测:

![image-20221123213535086](https://user-images.githubusercontent.com/82318011/203565769-3816b1dc-82fb-4952-a63d-4993b495a784.png)


metricbeat：收集指标数据,包括系统运行状态、CPU内存利用率，还可以收集nginx、redis、haproxy等服务的指标数据。

将metricbeat-8.5.1-amd64.deb包上传到一台业务服务器上

```shell
dpkg -i metricbeat-8.5.1-amd64.deb 

vim  /etc/metricbeat/metricbeat.yml

67   host: "169.254.90.140:5601"              #保证业务主机可以连接到kibana 主机
68   setup_kibana_username: "magedu"
69   setup_kibana_passwd: "123456"

96   hosts: ["169.254.90.140:9200"]          #保证业务主机可以连接到es集群任意一台主机

103   username: "elastic"
104   password: "123456"

systemctl enable metricbeat.service  --now
```

登录kibana web页面——>observability

![image-20221123204406459](https://user-images.githubusercontent.com/82318011/203566045-d2782648-9d8f-4fd7-aa82-93e977e1cd65.png)

![image-20221123204512702](https://user-images.githubusercontent.com/82318011/203566180-4db95d5f-cd01-4fca-acf9-f89ad9f483fc.png)
