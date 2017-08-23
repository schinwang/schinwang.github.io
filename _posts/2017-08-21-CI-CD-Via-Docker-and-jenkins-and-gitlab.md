---
bg: "meteor.jpg"
layout: post
title: "基于Jenkins gitlab docker的CI/CD"
summary: ""
tags: ['docker']
categories: docker
author: chason
---
##### 准备条件

1. 使用docker的机器，添加用户时需要指定用户的uid  ： 
   sudo groupadd -g 500 work && useradd -g 500 -u 500 work   
   否则可能出现容器无权限访问挂载数据卷的问题
2. OS  Requirements:
   [64bit-CentOs 7](https://download.docker.com/linux/centos/docker-ce.repo)
3. [Docker Installation](https://docs.docker.com/engine/installation/linux/centos/#install-using-the-repository)
   版本： CE（社区版）
   安装:    sudo yum install -y yum-utils device-mapper-persistent-data lvm2 &&  sudo yum-config-manager --add-repo [https://download.docker.com/linux/centos/docker-ce.repo](https://download.docker.com/linux/centos/docker-ce.repo) && sudo yum makecache fast && sudo yum -y install docker-ce && sudo systemctl start docker 
4. 建立用户组（使用work账号管理）
   sudo groupadd docker  ; sudo usermod -aG docker work
5. sed -i   's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g'  /etc/sysctl.conf  && sysctl -p

##### Jenkins机器：

##### 镜像仓库

1. Harbor Installation [镜像仓库管理工具](https://github.com/vmware/harbor)
   sudo yum -y install wget && mkdir -p /data/soft && sudo chown -R work:work /data/soft && cd /data/soft && wget [https://github.com/vmware/harbor/releases/download/v1.1.0/harbor-online-installer-v1.1.0.tgz](https://github.com/vmware/harbor/releases/download/v1.1.0/harbor-online-installer-v1.1.0.tgz) && tar zxf harbor-online-installer-v1.1.0.tgz
   cd harbor && 编辑  harbor.cfg  && sh install.sh

示例：

```python
## Configuration file of Harbor

#The IP address or hostname to access admin UI and registry service.
#DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname = harbor.***

#The protocol for accessing the UI and token/notification service, by default it is http.
#It can be set to https if ssl is enabled on nginx.
ui_url_protocol = https

#The password for the root user of mysql db, change this before any production use.
db_password = ******

#Maximum number of job workers in job service  
max_job_workers = 3 

#Determine whether or not to generate certificate for the registry's token.
#If the value is on, the prepare script creates new root cert and private key 
#for generating token to access the registry. If the value is off the default key/cert will be used.
#This flag also controls the creation of the notary signer's cert.
customize_crt = on

#The path of cert and key files for nginx, they are applied only the protocol is set to https
ssl_cert = /etc/ssl/private/**.pem
ssl_cert_key = /etc/ssl/private/**.key

#The path of secretkey storage
secretkey_path = /data

#Admiral's url, comment this attribute, or set its value to to NA when Harbor is standalone
admiral_url = NA

#NOTES: The properties between BEGIN INITIAL PROPERTIES and END INITIAL PROPERTIES
#only take effect in the first boot, the subsequent changes of these properties 
#should be performed on web ui

#************************BEGIN INITIAL PROPERTIES************************

#Email account settings for sending out password resetting emails.

#Email server uses the given username and password to authenticate on TLS connections to host and act as identity.
#Identity left blank to act as username.
email_identity = 

email_server = smtp.qq.com
email_server_port = 587
email_username = **@foxmail.com
email_password = ********
email_from = harbor <**@foxmail.com>
email_ssl = true

##The initial password of Harbor admin, only works for the first time when Harbor starts. 
#It has no effect after the first launch of Harbor.
#Change the admin password from UI after launching Harbor.
harbor_admin_password = *******

##By default the auth mode is db_auth, i.e. the credentials are stored in a local database.
#Set it to ldap_auth if you want to verify a user's credentials against an LDAP server.
#auth_mode = db_auth
auth_mode = ldap_auth

#The url for an ldap endpoint.
ldap_url = ldaps://***

#A user's DN who has the permission to search the LDAP/AD server. 
#If your LDAP/AD server does not support anonymous search, you should configure this DN and ldap_search_pwd.
#ldap_searchdn = uid=searchuser,ou=people,dc=mydomain,dc=com

#the password of the ldap_searchdn
#ldap_search_pwd = password

#The base DN from which to look up a user in LDAP/AD
ldap_basedn = ou=people,dc=mydomain,dc=com

#Search filter for LDAP/AD, make sure the syntax of the filter is correct.
#ldap_filter = (objectClass=person)

# The attribute used in a search to match a user, it could be uid, cn, email, sAMAccountName or other attributes depending on your LDAP/AD  
ldap_uid = uid 

#the scope to search for users, 1-LDAP_SCOPE_BASE, 2-LDAP_SCOPE_ONELEVEL, 3-LDAP_SCOPE_SUBTREE
ldap_scope = 3 

#Timeout (in seconds)  when connecting to an LDAP Server. The default value (and most reasonable) is 5 seconds.
ldap_timeout = 5

#Turn on or off the self-registration feature
self_registration = off

#The expiration time (in minute) of token created by token service, default is 30 minutes
token_expiration = 30

#The flag to control what users have permission to create projects
#Be default everyone can create a project, set to "adminonly" such that only admin can create project.
project_creation_restriction = everyone

#Determine whether the job service should verify the ssl cert when it connects to a remote registry.
#Set this flag to off when the remote registry uses a self-signed or untrusted certificate.
verify_remote_cert = on
#************************END INITIAL PROPERTIES************************
#############
```

##### php基础镜像构建

Dockerfile:

    ```shell
FROM centos:centos6.6
MAINTAINER chason
ADD bfrontapi.tar.gz /data/deploy/
COPY run.sh /data/deploy/bfrontapi/
RUN groupadd -g 500 work && \
    useradd -u 500 -g 500 work && \
    ln -s /data/deploy /opt/deploy && \
    ln -s /lib64/libpcre.so.0.0.1 /lib64/libpcre.so.1 && \
    chown work:work /data/deploy/bfrontapi/run.sh && \
    chmod u+x /data/deploy/bfrontapi/run.sh && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
USER work
WORKDIR /data/deploy/bfrontapi/
VOLUME /data/deploy/bfrontapi/log
    ```

run.sh

```shell
#!/bin/sh
/data/deploy/bfrontapi/hhvm/bin/hhvm_control start
sh /data/deploy/bfrontapi/webserver/loadnginx.sh start
trap "sh /data/deploy/bfrontapi/webserver/loadnginx.sh stop; exit" TERM
while true
do 
    sleep 5
done
```



##### php持续构建（tag-push trigger)build.sh

```shell
docker login -u username -p passwd https://harbor.***       
docker pull harbor***       
git_tag=git tag|tail -1
URL='harbor.***'
TAG=$URL/php-api/php-api:$git_tag
docker build -t $TAG ./docker/.
docker push $TAG
docker rmi $TAG Dockerfile
```

```sehll
FROM harbor.qyvideo.net/php-api/php-api:basic       
MAINTAINER chason
ADD php-app.tgz /data/deploy/bfrontapi
RUN sh /opt/deploy/bfrontapi/deployconf.sh /opt/deploy/bfrontapi production restart && \
    mkdir -p /home/work/odp/log && \
    touch /home/work/odp/log/error.log
EXPOSE 8000
CMD /data/deploy/bfrontapi/run.sh
```



##### jenkins 触发器设置

1. 安装gitlab插件
   ![jenkins_gitlab_plugin](/assets/images/posts/20170821/jenkins_gitlab_plugin.png)
2. gitlab webhook 添加
   ![gitlab_webhook](/assets/images/posts/20170821/gitlab_webhook.png))

##### 其他        

1. [Docker Compose Installation](https://docs.docker.com/compose/install/)
   sudo -i;
   curl -L https://github.com/docker/compose/releases/download/1.12.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose  && chmod +x /usr/local/bin/docker-compose
    which docker-compose （如果没有的话，把 /usr/local/bin/加到路径里面） 
2. swarm (docker 集群管理工具）

##### 常见问题

1. yum 安装报Rpmdb checksum is invalid错误
   尝试yum install ** || yum install **
   或yum clean allDocker 
2. 镜像size太大
   Docker file 尽量不要使用yum update（请尽量用最新的镜像代替改命令）chown命令的目标对象如果很大的话，可以在加入docker镜像之前提前chown
