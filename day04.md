一、部署jenkins master及多slave环境

1.两台slave节点都要安装openjdk-11-jdk、git、ntp等软件

​    apt install openjdk-11-jdk、git、ntp(保证master节点与node节点的时间同步)

​     crontab -e

​    */5 * * * *    /usr/bin/ntpdate   time1.aliyun.com &> /dev/null  && /usr/sbin/hwclock  -w

 2.登录jenkens web页面系统管理------>添加节点

![image-20221116195316526](C:\Users\wzw18\Desktop\day03 图片\image-20221116195316526.png)

 3.添加节点信息，远程工作目录要与master节点一致

![image-20221117102250315](C:\Users\wzw18\Desktop\day03 图片\image-20221117102250315.png)

4用username with password 方式加入jenins集群

![image-20221116200331999](C:\Users\wzw18\Desktop\day03 图片\image-20221116200331999.png)

![image-20221116200351996](C:\Users\wzw18\Desktop\day03 图片\image-20221116200351996.png)

二、基于jenkins视图对jenkins job进行分类

jenkins视图分为列表视图(对项目进行分类项目)、包括全局视图(所有视图)和我的视图(用户登录后看到自己所在项目的视图，即有权限看到的视图)

可以用正则匹配项目视图

![image-20221117101459136](C:\Users\wzw18\Desktop\day03 图片\image-20221117101459136.png)

![image-20221117101520612](C:\Users\wzw18\Desktop\day03 图片\image-20221117101520612.png)

三.总结jenkins pipline基本语法

流水线过程定义在 Pipeline{}块中，在Pipeline 块定义了整个流水线中完成的所有的操作，pipeline 分为stage阶段和step步骤。Stage阶段，一个pipline可以划分为若干个stage，每个stage都是一个操作阶段，比如代码clone、代码编译、代码测试和代码部署，阶段是一个逻辑分组，在pipline中可以实现跨多个node执行不同的stage。Step步骤，step是jenkins pipline最基本的操作单元，一个stage中可以有多个step，例如在代码clone的stage中需要定义代码clone的step、在代码编译stage需要定义代码编译的step。Node是jenkins工作节点，可以是jenkins master也可以是jenkins slave，node是执行step的具体服务器。

1.基于agent指令选择jenkins节点。agent any : 表示可以在任何可用的节点执行，由jenkins自动分配。

`pipeline {`

`agent any`

`}`

none表示pipline脚本没有定义在默认执行的jenkins节点，需要在后续的每一个stage中单独定义节点

`pipeline {`

  `agent none`

  `stages {`

​    `stage('代码clone'){`

​      `agent any`

​        `steps{`

​          `sh 'echo 代码clone'`

​        `}`

  `}`

  `stage('代码部署'){`

​    `agent any`

​      `steps{`

​        `sh 'echo 代码部署'`

​      `}`

  `}`

`}`

`}`

2.label表示通过标签指定在指定的节点执行从操作

`pipeline{`

 `//agent any //全局必须带有agent,表明此pipeline执行节点`

 `agent { label 'jenkins-node1' } //基于label指定具体执行的步骤节点，非必须`

  `stages{`

   `stage("代码clone"){`

​    `//#agent { label 'master' } //基于label指定具体执行的步骤节点，非必须`

​    `steps{`

​     `sh "cd /var/lib/jenkins/workspace/pipline-test1 && rm -rf ./*"`

​     `git branch: 'main', credentialsId: '9de167f4-0359-467d-85ba-acabd48c0f23', url: 'git@172.31.5.101:magedu/app1.git'`

​     `echo "代码 clone完成"`

`}`

`}`

   `stage("代码构建"){`

​    `//#agent { label 'master' } //基于label指定具体执行的步骤节点，非必须`

​    `steps{`

​     `sh "cd /var/lib/jenkins/workspace/pipline-linux40-app1-develop && tar czvf linux40.tar.gz ./*"`

  `}`

  `}` 

 `}`

}

3. node和 label 的功能类似都是用于指定节点，区别是node可以额外设置一些参数配置，比如设置customWorkspace(设置当前stage的自定义工作目录)

`pipeline {`

  `agent none`

​    `stages {`

​      `stage('代码clone'){`

​        `agent {`

​          `node {`

​            `label 'jenkins-master'`

​           `customWorkspace "/data/gitdata/magedu"`

​    `}`

`}`

​          `steps {`

​            `sh "echo 代码clone"`

  `}`

`}`

​        `stage('代码部署'){`

​           `agent {`

​            `node {`

​               `label 'jenkins-master'`

​              `customWorkspace "/data/gitdata/magedu"`

  `}`

`}`

​           `steps {`

​              `sh "echo 代码部署"`

  `}`

  `}`

  `}`

`}`

4. 基于input实现交互式操作：

   Input 指令可以在流水线中实现必要的交互式操作，比如选择要部署的环境、是否向后继续执行任务等。input配置简介：

   message：必选，在input界面的提示信息，比如：“是否继续？”等，内容可自定义，

   id：可选，input 的标识符，默认为 stage 的名称。

   ok：可选，确认按钮的显示信息，比如可以是“确定” 、 “允许”等 自定义内容，默认继续为Proceed 、取消为 Abort。

   submitter：可选，允许提交 input 操作的用户或组的名称，如果为空，任何登录用户均可提交 input。parameters：提供一个参数列表供 input 使用。

`pipeline {`

  `agent any`

​    `stages {`

​      `stage('交互测试') {`

​        `input {`

​          `message "是否继续部署?"`

​          `ok "继续部署"`

​          `submitter "jenkinsadmin"`

   `}`

​       `steps {`

​          `echo "Hello jenkins!"`

​    `}`

  `}`

 `}`

`}`

5. post指令

   post一般用于pipline流水线执行后的进一步处理，比如错误通知等，post可以针对流水线不同的执行结果做出不同的处理，比如执行成功做什么处理、执行失败做什么处理等。

   Post 可以定义在 Pipeline 或 stage 中，目前支持以下条件：

   always：无论Pipeline或stage的最后是执行成功还是失败,都执行post中定义的指令。

   changed：只有当前Pipeline或stage的完成状态与它之前的运行不同时,比如从成功转换为失败、或从失败转换为成功,才执行post中定义的指令。

   fixed：当本次Pipeline或stage成功,且上一次构建是失败或不稳定时,就执行post中定义的指令,从失败转换为成功

   regression：当本次Pipeline或stage的状态为失败、不稳定或终止,且上一次构建的状态为成功时,就执行post中定义的指令,从成功转换为失败。

   failure：只有当前Pipeline或stage的完成状态为失败(failure),才允许在post部分运行该步骤,而且通常这时在Web界面中显示为红色。

   success：当前执行状态为成功(success),执行post步骤,通常在Web界面中显示为蓝色或绿色。

   unstable：当前状态为不稳定(unstable),执行post步骤,通常原因是由于测试失败或代码违规等造成,而且会在Web界面中显示为黄色。

   aborted：当前状态为终止(aborted),执行该post步骤,通常由于流水线被手动终止触发,这时在在Web界面中显示为灰色。

   unsuccessful：当前状态只要不是success时,执行该post步骤；

   cleanup：无论pipeline或stage的完成状态如何,都允许运行该post中定义的指令,和always的区别在于cleanup会在post其它条件执行之后执行(最后执行cleanup)。

   pipline-job-test9 演示在这执行异常后post阶段的操作，cleanup会晚于always执行，发送邮件需要提前配置好jenkins的邮件通知配置：

   `pipeline {`

     `agent any`

   ​    `stages {`

   ​      `stage('post测试-代码clone阶段') {`

   ​        `steps {`

   ​          `sh 'echo git clone'`

   ​          `sh 'cd /data/xxx' //此步骤会执行失败，用于验证构建失败的邮件通知`

   ​        `}`

   ​        `post {`

   ​        `cleanup {`

   ​          `echo "post cleanup-->测试是否执行"`

   ​        `}`

   ​        `always {`

   ​          `echo "post always"`

   ​        `}`

   ​        `aborted {`

   ​          `echo "post aborted"`

   ​        `}`

   ​        `success {`

   ​        `script {`

   ​          `mail to: '2973707860@qq.com' ,`

   ​        `subject: "Pipeline Name: ${currentBuild.fullDisplayName}" ,body: " ${env.JOB_NAME} -Build Number-${env.BUILD_NUMBER} - 构建成功!\n 点击链接${env.BUILD_URL} 查看详情"`

   ​      `}`

      `}`

   ​        `failure {`

   ​           `script {`

   ​             `mail to: '2973707860@qq.com' ,`

   ​                `subject: "Pipeline Name: ${currentBuild.fullDisplayName}" ,`

   ​                `body: " ${env.JOB_NAME} -Build Number-${env.BUILD_NUMBER} - 构建失败!\n 点击链接 ${env.BUILD_URL} 查看详情"`

   ​           `}`

   ​        `}`

   ​     `}`

      `}`

     `}`

   `}`

   6. pipeline是基于environment传递环境变量

   `pipeline {`

     `agent any`

     `environment { //全局的变量，在当前pipline所有的stage中都会生效`

   ​    `NAME='user1'`

   ​    `PASSWD='123456'`

     `}`

     `stages {`

   ​    `stage('环境变量stage1') {`

   ​      `environment { //定义在stage中的变量只会在当前stage生效，其他的stage不会生效`

   ​        `GIT_SERVER = 'git@172.31.5.101:magedu/app1.git'`

   ​      `}`

   ​      `steps {`

   ​        `sh """`

   ​          `echo '$NAME'`

   ​          `echo '$PASSWD'`

   ​          `echo '$GIT_SERVER'`

   ​        `"""`

   ​         `}`

   ​      `}`

   ​    `stage('环境变量stage2') {`

   ​      `steps {`

   ​        `sh """`

   ​        `echo '$NAME'`

   ​        `echo '$PASSWD'`

   ​        `"""`

   ​      `}`

   ​    `}`

     `}`

   `}`

7. pipeline基于parameters给step传递参数，可以基于parameters自定义参数，参数用于在执行pipline流水线的时候传递给step进行使用，比如传递选项参数、gitlab的分支、镜像仓库、镜像tag等信息。

   parameters简介：

   string： #字符串类型参数，可以传递账户名、密码等参数

   text： #文本型参数，一般用于定义多行文本内容的变量。

   booleanParam：#布尔型参数

   choice：#选择型参数，一般用于给定几个可选的值，然后选择其中一个进行赋值使用

   password： #密码型变量，一般用于定义敏感型变量，在 Jenkins 控制台会输出为*隐藏密码

`pipeline {`

  `agent any`

  `parameters {`

​    `string(name: 'BRANCH' , defaultValue: 'develop' , description: '分支选择') //字符串参数，会配置在jenkins的参数化构建过程中`

​    `choice(name: 'DEPLOY_ENV' , choices: ['develop' , 'production'], description: '部署环境选择') //选项参数，会配置在jenkins的参数化构建过程中`

`}`

  `stages {`

​    `stage('测试参数1') {`

​      `steps {`

​        `sh "echo $BRANCH"`

​      `}`

  `}`

​    `stage('测试参数2') {`

​      `steps {`

​        `sh "echo $DEPLOY_ENV"`

​      `}`

   `}`

  `}`

`}`

8. pipeline 基于if指令实现流程判断

   `pipeline {`

     `agent any`

     `environment {`

   ​    `//代码仓库变量`

   ​    `def BRANCH_NAME = 'main'`

     `}`

     `stages {`

   ​    `stage('if指令测试') {`

   ​      `steps {`

   ​        `script {`

   ​          `if (env.BRANCH_NAME == 'main') {`

   ​              `echo '我是master'`

   ​          `} else if (env.BRANCH_NAME == 'develop'){`

   ​              `echo '我是develop'`

   ​          `} else {`

   ​              `echo '我是默认的'`

   ​         `}`

   ​      `}`

   ​    `}`

     `}`

    `}`

   `}`

9. pipeline options选项配置参数简介

   buildDiscarder(logRotator(numToKeepStr: '5')) //保留5个历史构建版本

   timeout(time: 5, unit: 'MINUTES') //定义任务执行超时时间1小时，如果不加unit参数默认时间单位为5分钟，支持NANOSECONDS，MICROSECONDS，MILLISECONDS，SECONDS，MINUTES，HOURS，DAYS

​       timestamps() //在控制台显示命令执行的时间，格式为10:58:39

​       retry(2) //流水线构建失败后重试次数为2次

`pipeline {`

  `agent any`

  `environment { //全局的变量，在当前pipline所有的stage中都会生效`

​    `NAME='user1'`

​    `PASSWD='123456'`

  `}`

  `options { //定义pipline参数`

​    `buildDiscarder(logRotator(numToKeepStr: '5')) //保留5个历史构建版本`

​    `timeout(time: 5, unit: 'MINUTES') //定义任务执行超时时间1小时，如果不加unit参数默认时间单位为分钟，支持NANOSECONDS，MICROSECONDS，MILLISECONDS，SECONDS，MINUTES，HOURS，DAYS`

​    `timestamps() //在控制台显示命令执行的时间，格式为10:58:39`

​    `retry(2) //流水线构建失败后重试次数为2次`

  `}`

  `stages {`

​    `stage('代码clone') {`

​      `environment { //定义在stage中的变量只会在当前stage生效，其他的stage不会生效`

​        `GIT_SERVER = 'git@172.31.5.101:magedu/app1.git'`

​      `}`

​      `steps {`

​        `sh """`

​          `echo '代码clone'`

​          `"""`

​        `}`

​      `}`

​    `}`

`}`

四、部署代码质量检测服务sonarqube

sonarqube依赖postgresql 需要先部署postgresql

apt update  && apt install -y  postgresql

1. PostgreSQL环境初始化：

pg_createcluster --start 14 mycluster  #指定版本为PostgreSQL 14

vim /etc/postgresql/14/mycluster/pg_hba.conf

96 # IPv4 local connections:

97 host all all 0.0.0.0/0 scram-sha-256 #允许远程连接

vim /etc/postgresql/14/mycluster/postgresql.conf

60 listen_addresses = '*'     #defaults to 'localhost'; use '*' for all  

lsof -i:5432  # PostgreSQL端口验证：

2. 创建数据库及账户授权：

su - postgres   #切换到postgres普通用户

psql -U postgres   #进入到postgresql命令行敞口

postgres=# CREATE DATABASE sonar;  #创建sonar数据库

postgres=# CREATE USER sonar WITH ENCRYPTED PASSWORD '123456'; #创建sonar用户密码为123456

postgres=# GRANT ALL PRIVILEGES ON DATABASE sonar TO sonar; #授权用户访

postgres=# ALTER DATABASE sonar OWNER TO sonar; #执行变更

postgres=# \q #退出

3. 准备SonarQube Server 8.9.x环境

安装jdk 11  apt install -y openjdk-11-jdk

内核参数优化：vim /etc/sysctl.conf

vm.max_map_count = 262144

fs.file-max = 65536

ulimited 参数调整

vim /etc/security/limits.conf

`root                soft    core            unlimited`
`root                hard    core            unlimited`
`root                soft    nproc           1000000`
`root                hard    nproc           1000000`
`root                soft    nofile          1000000`
`root                hard    nofile          1000000`
`root                soft    memlock         32000`
`root                hard    memlock         32000`
`root                soft    msgqueue        8192000`
`root                hard    msgqueue        8192000`

*`soft    core            unlimited`

*`hard    core            unlimited`

*`soft    nproc           1000000`

*`hard    nproc           1000000`

*`soft    nofile          1000000`

*`hard    nofile          1000000`

*`soft    memlock         32000`

*`hard    memlock         32000`

*`soft    msgqueue        8192000`

*`hard    msgqueue        8192000`

4. 部署SonarQube 8.9.x：

mkdir /apps && cd /apps/

unzip sonarqube-8.9.10.61524.zip

ln -sv /apps/sonarqube-8.9.10.61524 /apps/sonarqube

 useradd -r -m -s /bin/bash sonarqube && chown sonarqube.sonarqube /apps/ -R && su - sonarqube

$ vim /apps/sonarqube/conf/sonar.properties

18 sonar.jdbc.username=sonar

19 sonar.jdbc.password=123456

37 sonar.jdbc.url=jdbc:postgresql://172.31.5.106/sonar

~$ /apps/sonarqube/bin/linux-x86-64/sonar.sh start

报错查看日志

tail /apps/sonarqube/logs/*.log

部署成功后用本机9000端口到浏览器访问



五、基于命令、shell脚本和pipline实现代码质量检测

部署sonar-scanner扫描器

官网下载sonar-scanner-cli-4.7.0.2747.zip 包并解压

unzip sonar-scanner-cli-4.7.0.2747.zip

ln -sv /apps/sonar-scanner-4.7.0.2747 /apps/sonar-scanner

vim /apps/sonar-scanner/conf/sonar-scanner.properties #修改sonar-scanner配置文件

\#----- Default SonarQube server

sonar.host.url=http://169.254.90.138:9000

#----- Default source code encoding

sonar.sourceEncoding=UTF-8

在sonar-scanne节点测试代码质量扫描

cat sonar-project.properties

\# Required metadata

 sonar.projectKey=magedu-python //#项目key,项目唯一标识、通常使用项目名称

sonar.projectName=magedu-python-app1 #项目名称，当前的服务名称

 sonar.projectVersion=1.0 #当前的代码版本



# Comma-separated paths to directories with sources (required)

sonar.sources=./src #源代码路径、sonar-scanner会扫描./src 下的代码



# Language

 sonar.language=py #编程语言类型



# Encoding of the source files

 sonar.sourceEncoding=UTF-8 #字符集

命令扫描：/apps/sonar-scanner/bin/sonar-scanner

基于传递参数扫描：

/apps/sonar-scanner/bin/sonar-scanner -Dsonar.projectKey=magedu -Dsonar.projectName=magedu-python-app1 -Dsonar.projectVersion=0.1 -Dsonar.sources=./src -Dsonar.language=py -Dsonar.sourceEncoding=UTF-8

基于pipepine扫描

pipeline {

  agent any

  parameters {

​    string(name: 'BRANCH' , defaultValue: 'develop' , description: '分支选择') //字符串参数，会配置在jenkins的参数化构建过程中

​    choice(name: 'DEPLOY_ENV' , choices: ['develop' , 'production'], description: '部署环境选择') //选项参数，会配置在jenkins的参数化构建过程中

}

  stages {

​    stage('变量测试1') {

​      steps {

​        sh "echo $env.WORKSPACE" //JOB的工作目录,可用于后期目录切换

​        sh "echo $env.JOB_URL" //JOB的URL

​        sh "echo $env.NODE_NAME" //节点名称，master 名称显示built-in

​        sh "echo $env.NODE_LABELS" //节点标签

​        sh "echo $env.JENKINS_URL" //jenkins的URL地址

​        sh "echo $env.JENKINS_HOME" //jenkins的家目录路径

​     }

  }

   stage('python源代码质量扫描') {

​      steps {

​        sh "cd $env.WORKSPACE && /apps/sonar-scanner/bin/sonar-scanner -Dsonar.projectKey=magedu -Dsonar.projectName=magedu-python-app1 -Dsonar.projectVersion=1.0 -Dsonar.sources=./src -Dsonar.language=py -Dsonar.sourceEncoding=UTF-8"

​      }

​    }

  }

}

扩展：jenkins安装Sonarqube Scanner插件、配置sonarqube server地址、基于jenkins配置代码扫描参数实现代码质量扫描    Execute SonarQube Scanner

1.在jenkins 系统管理--->插件管理安装sonarqube scanner for jenkins 2.14 插件，重启jenkins

2.在jenkins 系统管理--->全局工具配置SonarQube Scanner

3.在jenkins 系统管理--->系统管理---> SonarQube servers 配置sonar-server的名称和IP地址

4.新建magedu-app2_deploy-test2任务，build steps Analysis properties配置sonar-project.properties 项目配置文件，然后配置git 仓库拉取代码，立即构建



