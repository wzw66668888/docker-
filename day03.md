一、基于docker-compose实现对nginx+tomcat web 服务的单机编排

1. 安装docker-compose

   (1)apt 安装： 

      apt install docker-compose

      apt install python3-pip

      pip3 install docker-compose

    (2) 二进制安装：

   ​       到官网下载docker-compose https://github.com/docker/compose 并上传到本地

   ​       chmod a+x  docker-compose-x86_64   

   ​       cp docker-compose-Linux-x86_64 /usr/bin/docker-compose

​               docker-compose version  查看docker-compose版本

   2. 构建nginx和tomcat 业务镜像

        (1) 构建tomcat 业务镜像

         首先从官方下载ubuntu:v22.04的官方镜像,创建system-base镜像目录和dockerfile
         mkdir -p /opt/20221108/system-base
         `cat > dockerfile << EOF`

         `FROM ubuntu:22.04`
         `MAINTAINER "star 2973707860@qq.com"`

         `RUN apt update && apt install -y  iproute2 ntpdate tcpdump telnet traceroute nfs-kernel-   server nfs-common lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate   tcpdump telnet    traceroute gcc openssh-server lrzsz tree openssl libssl-dev libpcre3   libpcre3-dev zlib1g-dev ntpdate    tcpdump telnet traceroute iotop unzip zip make`
        `EOF`

        `#构建ubuntu-base镜像并保证镜像能正常运行`
        `docker build -t harbor.magedu.net/myserver/ubuntu-base:22.04  .   
        docker run -it --rm  harbor.magedu.net/mtserver/ubuntu-base:22.04` 

      (2)构建tomcat镜像

      mkdir -p /data/dockerfile/tomcat/tomcat-base

      cd  /data/dockerfile/tomcat/tomcat-base

      从官网下载apache-tomcat-9.0.21.tar.gz   jdk1.8.0_221.tar.gz

​        vim dockerfile

​         `FROM ubuntu-base:22.04` 

​         `MAINTAINER "star 2973707860@qq.com"`

​         `ADD jdk1.8.0_221.tar.gz /usr/local/src`

​         `ADD apache-tomcat-9.0.21.tar.gz /usr/local/src`

​         `ENV MYPATH /usr/local/src`

​         `WORKDIR $MYPATH`

​         `ENV  JAVA_HOME/usr/local/jdk1.8.0_221`

​         `ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar`

​         `ENV CATALINA_HOME /usr/local/apache-tomcat-9.0.21`

​         `ENV CATALINA_BASH /usr/local/apache-tomcat-9.0.21`

​         `ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin`

​         `EXPOSE 8080`

​         `CMD   /usr/local/src/apache-tomcat-9.0.21/bin/startup.sh && tail -F   /usr/local/apache-tomcat-9.0.2/bin/logs/catalina.out  #让tomcat在前台运行，保证容器不退出`

​        docker build -t harbor.magedu.net/myserver/tomcat:9.0.21  . 

​    （3）构建 nginx镜像

​      mkdir -p /opt/20221108/nginx

​      vim dockerfile  

​        FROM  ubuntu-base:22.04

​        ADD nginx-1.22.1.tar.gz  /usr/local/src/

​        RUN cd /usr/local/src/nginx-1.22.1 && ./configure  --prefix=/apps/nginx && make && make install &&    ln -sv /apps/nginx/sbin/nginx  /usr/bin

​        RUN  groupadd  -g 2088 nginx  && useradd -g nginx -s /usr/sbin/nologin -u 2088 nginx && chown -R nginx.nginx /apps/nginx

​        ADD nginx.conf  /apps/nginx/conf/   #添加location，访问动态nginx就转发到tomcat处理

​        ADD frontend.tar.gz  /apps/nginx/html

​        ENTRYPOINT ["/apps/nginx/sbin/nginx","-g","daemon off;"]

​        docker build -t harbor.magedu.net/myserver/nginx-apline:1.22.1  . 

​      (4)  使用docker-compose 构建nginx+tomcat web 服务的单机编排

​        vim  docker-compose.yml

​        `version:  '3.8'`

​        `service:`

​            `nginx-server:`

​                `image: nginx-apline:1.22.1`

​                `container_name:  nginx-web1`

​                `expose:`

​                    `-  80`

​                    `-  443`

​                 `ports:` 

​                     `- "80:80"`

​                     `- "443:443"`

​                 `networks:` 

​                      `- front`

​                      `- backend`

​                 `links:`

​                    `-  tomcat-server`

​             `tomcat-server:`

​                  `image: tomcat:9.0.21` 

​                   `container_name:  tomcat-app1`

​                   `expose:`

​                       `-  8080`

​                   `ports:`

​                        `- "8080:8080"`

​                   `networks:`

​                        `- backend`

​       `networks:`

​            `front:`

​                `driver: bridge`

​             `backend:`

​                 `driver: bridge`

​              `default:`

​                   `external:`

​                        `name: brodge`

docker-compose  up   -d       

docker-compose  ps   #查看容器是否已经运行



二.安装gitlab、创建group、user和project并授权

清华下载 gitlab-ce_15.4.3-ce.0_amd64.deb 上传至gitlab服务器

dpkg -i  gitlab-ce_15.4.3-ce.0_amd64.deb  安装gitlab服务器

修改默认配置文件

vim /etc/gitlab/gitlab.rb

external_url 'http://169.254.90.132'    #external 修改为自己的gitlab地址

gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.qq.com"                 #如果是163邮箱应改为“smtp.163.com”
gitlab_rails['smtp_port'] = 465 
gitlab_rails['smtp_user_name'] = "2968687730@qq.com"
gitlab_rails['smtp_password'] = "********"                             #修改为自己的邮箱登录密码
gitlab_rails['smtp_domain'] = "qq.com"                          
gitlab_rails['smtp_authentication'] = :login
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['gitlab_email_from'] = "2968687730@qq.com"
user["git_user_email"] = "2968687730@qq.com"

gitlab-ctl reconfigure   #重新配置服务

gitlab 初始登录账户为root  密码在/etc/gitlab/initial_root_password 文件

创建项目组

![image-20221109113609646](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109113609646.png)

创建用户

![image-20221109114008737](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109114008737.png)

修改密码

![image-20221109114341205](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109114341205.png)

创建项目

![image-20221109114624984](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109114624984.png)



账户权限分类：

Guest-访客，可以创建issue、发表评论，不能读写版本库

Reporter-Git项目测试人员，可以克隆代码，不能提交，QA、PM可以赋予这个权限

Developer-Git项目开发人员，可以克隆代码、开发、提交、push，RD(Research and Development engineer,研发工程师)可以赋予此权限

Master-Git项目管理员，可以创建项目、添加tag、保护分支、添加项目成员、编辑项目，核心RD负责人可以赋予此权限

Owner-Git系统管理员即Administrator，可以设置项目访问权限、删除项目、迁移项目、管理组成员，研发组leader可以赋予此权限

将user1用户授权为Developer

![image-20221109141146534](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109141146534.png)

将user2用户授权为owner

![image-20221109141320584](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109141320584.png)

![image-20221109141438232](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109141438232.png)

三、熟练git命令的基本使用，通过git命令实现源代码的clone、push等基本操作

登录一台机器将上面创建的app1的项目clone

 git clone http://169.254.90.132/magedu/app1.git

git add .                          # 添加当前目录下所以变化的内容到暂存区

git commit  -m  "xxx"   # 将代码提交到本地仓库 

git  push                         # 将代码上传到gitlab 服务器

四、熟练掌握对gitlab服务的数据备份与恢复

首先git pull  拉取最新代码

gitlab-ctl stop  unicorn  sidekip    # 停止gitlab写入

gitlab-rake  gitlab:backup:create

备份/etc/gitlab/gitlab.rb (配置文件)、/etc/gitlab/gitlab-secret.json (key文件)和/var/opt/gitlab/nginx/conf (nginx配置文件)

gitlab-ctl start unicorn sidekiq

模拟删除user2 用户

gitlab-ctl stop  unicorn  sidekip 

gitlab-rake gitlab:backup:restore  BACKUP=1667984220_2022_11_09_15.4.3

gitlab-ctl start unicorn sidekiq

到gitlab前端验证数据是否已经恢复

五、部署jenkins服务器并安装gitlab插件、实现代码免秘钥代码clone

清华源下载jenkins镜像  jenkins_2.361.3_all.deb 上传至jenkins服务器

 安装jdk环境  apt install openjdk-11-jdk    apt install  net-tools(jenkens 依赖net-tools镜像)

 dpkg -i  jenkins_2.361.3_all.deb  &&  systemctl stop jenkins

 修改jenkins的service文件

  vim /lib/systemd/system/jenkins.service

   JENKINS_USER=root(提升jenkins权限为root权限)

   JENKINS_GROUP=root

   Environment="JAVA_OPTS=-Djava.awt.headless=true -   Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true"

  systemctl daemon-reload 

  systemctl start jenkins.service 

  ps -ef |grep jenkins (查看是否是root权限,配置是否正确)

  cat /var/lib/jenkins/secrets/initialAdminPassword  查看密钥，根据前端提示登录

  安装gitlab 插件

![image-20221109211654916](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109211654916.png)

  安装后重启jenkins  systemctl restart jenkins

  jenkins代码免密clone

  jenkins 生成密钥对并拷贝公钥至jenkins服务器

  ssh-keygen   ssh-copy-id  169.254.90.132  

  登录gitlab web页面添加ssh 密钥

![image-20221109223252609](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109223252609.png)

登录jenkins服务器 git clone git@169.254.90.132/magedu/app1.git 测试

mkdir -p  /data/gitdata/magedu   

vim test1.sh

#!/bin/bash
cd /data/gitdata/magedu
rm -rf app1
git clone ssh://git@169.254.90.132:36000/magedu/app1.git

登录jenkins web页面新建任务并执行立即构建验证结果

![image-20221109225442103](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109225442103.png)

![image-20221109225509623](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109225509623.png)

![image-20221109225638563](C:\Users\wzw18\AppData\Roaming\Typora\typora-user-images\image-20221109225638563.png)

