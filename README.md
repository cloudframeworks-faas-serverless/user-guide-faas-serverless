# [云框架]FaaS

(简介)

# 内容概览

* [在线演示](#在线演示)
* [快速部署](#快速部署)
* * [镜像部署](#镜像部署)
* [框架说明](#框架说明) 
   * [业务](#业务)
      * [业务背景](#业务背景)
      * [业务架构](#业务架构)
      * [业务模块](#业务模块)
   * [组件](#组件)
      * [组件架构](#组件架构)
      * [组件1](#组件1)
      * [组件2](#组件2)
      * [组件3](#组件3)
      * [组件4](#组件4)
      * [组件5](#组件5)
* [如何变成自己的项目](#如何变成自己的项目)
* [生产环境](#生产环境)
* [常见问题](#常见问题)
* [更新计划](#更新计划)
* [参与贡献](#参与贡献)
* [加入社群](#加入社群)



# 快速部署

## 镜像部署

1. 系统环境准备

   系统基于[Docker](https://docker.io)部署和运行，我们需要为我们的系统安装最新版的Docker服务。

   * Linux系统

     1. 国内网络环境

         ```
         curl -sSL https://get.daocloud.io/docker | sh
         ```

     2. 国外网络环境

        ```
        curl -s https://get.docker.com/ | sudo sh
        ```

   * MacOS

     请参考[https://docs.docker.com/docker-for-mac/](https://docs.docker.com/docker-for-mac/)

2. 基于镜像安装组件

     这里我们使用faas.pro域名为例进行说明。你需要使用你的域名更换类似`traefik.frontend.rule=Host:api.faas.pro`中的域名指定。

*    安装数据持久化服务MYSQL

     ```
     docker run -d --restart=always -v `pwd`/data:/var/lib/mysql \
            --name function-mysql \
            --restart=always  \
            -e MYSQL_DATABASE=func \
            -e MYSQL_USER=func \
            -e MYSQL_ROOT_PASSWORD=root-password\
            -e MYSQL_PASSWORD=func-password \
            mysql:5.5
     ```

* 安装消息队列服务REDIS

     ```
     docker run -d --name function-redis \
       --restart=always  \
       -v `pwd`/data:/data\
       redis redis-server --appendonly yes
     ```

* 安装API服务

     ```
     docker run -d --link function-mysql:db \
        --link function-redis:mq \
        --restart always\
        --name function \
        -l traefik.port=8080\
        -l traefik.tags=function-api \
        -l traefik.frontend.entryPoints=http \
        -l traefik.frontend.rule=Host:api.faas.pro \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -v $PWD/data:/app/data \
        -e DB_URL="mysql://func:func-password@tcp(db:3306)/func" \
        -e MQ_URL="redis://mq:6379" \
        hub.faas.pro/functions
     ```

* 安装UI控制台

     ```
     docker run -d --restart=always --name function-ui --link function:api \
           -e "API_URL=http://api:8080" \
           -l traefik.tags=function-ui \
           -l traefik.frontend.entryPoints=http \
           -l traefik.port=4000 \
           -l traefik.frontend.rule=Host:www.faas.pro\
           iron/functions-ui
     ```

* 安装镜像仓库服务

     ```
     docker run -d --name function-hub \
        --restart always \
        -v `pwd`/data:/var/lib/registry \
        -l traefik.port=5000\
        -l traefik.tags=function-hub\
        -l traefik.frontend.rule=Host:hub.faas.pro\
        -l traefik.protocol=http\
        -l traefik.frontend.entryPoints=https \
        registry:2
     ```

* 安装负载均衡和代理服务traefik

     编辑trafik的配置文件`traefik.toml`

     ```
     [entryPoints]
        [entryPoints.http]
        address = ":80"
        [entryPoints.https]
        address = ":443"
          [entryPoints.https.tls]
            [[entryPoints.https.tls.certificates]]
            CertFile = "log/hub.faas.pro/hub.faas.pro.pem"
            KeyFile = "log/hub.faas.pro/hub.faas.pro.key"

     traefikLogsFile = "log/traefik.log"
     accessLogsFile = "log/access.log"
     logLevel = "DEBUG"

     [docker]
     constraints = ["tag==function-*"]
     # Requiredi
     endpoint = "unix:///var/run/docker.sock"
     # Required
     domain = "faas.pro"
     watch = true
     exposedbydefault = true
     ```

     需要使用你的域名证书存储路径替换配置文件的证书路径（容器内路径）。下面启动容器时需要将证书挂载到容器中。此处证书服务主要是给镜像仓库使用。

     ```
     docker run -d -p 9999:8080 -p 80:80 -p 443:443\
          -v `pwd`/traefik.toml:/etc/traefik/traefik.toml\
          -v /var/run/docker.sock:/var/run/docker.sock\
          --restart always\
          -v `pwd`/log:/log\
          --name=traefik\
          traefik --web
     ```

   ​       上述步骤完成后访问`<你的域名或IP>:9999` 你将看到类似下图所示的服务：

​         ![](service.png)

   ​      访问`<你的域名或IP>`可以进入简单控制台：

​          ![](ui.png)

3. 安装客户端

   ```
   curl http://fs.faas.pro/fn | sh
   export API_URL=http://api.faas.pro
   fn --help
   ```


# 框架说明

* 业务
* 组件

   