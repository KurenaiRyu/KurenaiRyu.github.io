---
title: 将项目部署在docker中
date: 2019/3/14 11:39
categories: 
    - docker
tags: 
    - docker
    - ops
comments: true
thumbnail: /gallery/machi.png
---
docker中有两个概念，容器与镜像。 

镜像我们可以简单的理解为是装机用的光盘，而容器就是我们用光盘所装的一个系统。实际上我们就是由镜像生成一个容器，一个镜像可以生成多个容器（只要容器互不相冲突）。

而镜像的生成就是由配置文件来决定（配置文件参数可由后期运行指令等等操作更改），具体如何配置不详细说明。
<!--more-->
> Banner: {% link カントク - COLORS http://www.pixiv.net/member_illust.php?mode=medium&illust_id=47646872 カントク - COLORS %}

配置文件例子  
Dockerfile
``` dockerfile
#后端java项目

#基础镜像
FROM 192.168.1.2:5000/library/centos-jdk:1.7.79
#作者
MAINTAINER kurenai kurenai@moe.com
#执行命令，主要用来安装相关的软件
#RUN 
#添加文件
ADD target/supervise-svc-0.0.1-SNAPSHOT.jar /usr/local
RUN chmod u+x /usr/local/supervise-svc-0.0.1-SNAPSHOT.jar
#挂载目录到容器
#VOLUME ["/data"]
#环境变量设置
#ENV 
#开放端口
EXPOSE 1234
#启动时执行的命令
CMD ["/bin/bash"]
#启动时执行的命令
ENTRYPOINT ["java","-jar","-Xms2048m", "-Xmx2048m", "-XX:PermSize=256M", "-XX:MaxPermSize=256M","/usr/local/supervise-svc-0.0.1-SNAPSHOT.jar"]
```

``` dockerfile
#前端react项目

#基础镜像
FROM xxx.xxx.com:5000/library/ui-nginx:latest

#维护人信息
MAINTAINER kurenai kurenai@moe.com

#工作目录
WORKDIR /usr/local/nginx

ADD ui.tar.gz /usr/local/nginx/html/

ADD nginx.conf /usr/local/nginx/conf/

#暴露端口
EXPOSE 80

#连接时执行的命令
CMD ["/bin/bash"]

#启动时执行的命令
#ENTRYPOINT nginx -g "daemon off;"
ENTRYPOINT /opt/run.sh
```
- 基础命令

``` bash
docker ps       #查看doker中正在运行的容器列表
docker images   #查看docker中的镜像列表
docker build    #将当前目录下的文件打包为镜像
docker rm       #移除容器
docker rmi      #移除镜像
docker pull     #拉取镜像
docker logs -f  #查看日志
docker restart  #重启容器
docker stop     #停止容器运行
```

## docker-compose
能够比较集中的管理镜像和容器的部署问题，不必用像原生docker那样一个一个项目进行打包镜像生成容器，只需要把n个项目的配置写在配置文件中即可进行批量打包，拉取镜像，生成容器

配置文件例子
docker-compose.yml

``` yaml
version: '2.2'
services:
  #项目名称，用docker-compose做管理时，每个项目用这里配置的名称进行单独管理
  base:
    #镜像名，拉取镜像时也是用这个名字作为地址
    image: 192.168.1.2:5000/test/base-svc:0.0.1-SNAPSHOT 
    #打包路径，即docker build的路径
    build: /opt/dockerfile/base-svc
    restart: always
    #环境变量
    environment:
      defaultZone: http://192.168.1.3:1200/eureka/
    #开放端口
    ports: 
      - "1430:1430"
    #网络连接模式
    network_mode: "bridge"

  supervise:
    image: 192.168.1.2:5000/test/supervise-svc:0.0.1-SNAPSHOT
    build: /opt/dockerfile/supervise-svc
    restart: always
    environment:
      defaultZone: http://192.168.1.3:1200/eureka/
    ports: 
      - "1570:1570"
    network_mode: "bridge"
```
- 常用命令

``` bash
docker-copmpose build       #打包镜像，后面不加项目名则打包所有配置了build的项目，可接多个项目名，用空格隔开
docker-compose up -d        #后台运行项目，寻找本地镜像生成容器（若镜像更新则重启用新镜像生成容器），或者docker-compose.yml文件改变了也会进行更新容器，同样不接项目名为所有项目，也可以接多个项目名
docker-compose logs -f      #查看日志，同上可接项目名
docker-compose pull         #拉取镜像，同上可接项目名
docker-compose restart      #重启容器，同上
docker-compose stop         #停止运行容器，同上
```

## 实际生产环境部署项目

`/opt/dockerfile`文件夹下是每个项目的目录，每个目录下是一个Dockerfile配置文件+打包的项目文件（java为.jar, react为tar.gz, 视项目和配置文件而定）

`/opt/cloud`目录下是分的几个类，把几个项目归为一起，项目下是docker-compose.yml配置文件

以后端java项目`meeting-svc`为例 
1. 打包项目为jar包，
2. 将jar包放在`/opt/dockerfile/meeting-svc`目录下
3. 进入到`/opt/cloud/service`
4. 运行指令`docker-compose build meeting`(配置文件中配置的项目名为`meeting`)打包镜像
5. 运行`docker-compose up -d meeting`更新容器并运行
6. 用`docker-compose logs -f meeting`进行查看日志
---
ps： 后端打包为tar -zcvf xxx.tar.gz -C dist/ .      #dist为编译文件目录