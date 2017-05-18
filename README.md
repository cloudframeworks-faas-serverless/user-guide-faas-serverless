# [云框架]FaaS

(简介)

# 内容概览

* [在线演示](#在线演示)
* [快速部署](#快速部署)
* * [镜像部署](#镜像部署)
   * [应用开发与部署](#部署应用方法)
* [框架说明](#框架说明) 
   * [业务](#业务)
      * [业务背景](#业务背景)
   * [组件](#组件)
      * [组件架构](#架构图)
      * [Traefik](#Traefik)
      * [FunctionAPI](#FunctionAPI)
      * [Mysql](#Mysql)
      * [Redis](#Redis)
      * [Fn](#Fn)
      * [Hub](#Hub)
* [如何变成自己的项目](#如何变成自己的项目)
* [生产环境](#生产环境)
* [常见问题](#常见问题)
* [更新计划](#更新计划)
* [参与贡献](#参与贡献)
* [加入社群](#加入社群)




# 在线演示

​      移步:   http://www.faas.pro


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


 ## 部署应用方法

### 实例一：获取twitter指定用户的最近推文

开始前，请确认你已完成以上步骤。若你想要先体验部署应用，可使用本框架维护服务端：http://api.faas.pro

1. 查看源码

   (GitHub地址)[https://github.com/cloudframeworks-functionservice/function-example/tree/master/twitter]

2. 构建与运行应用

   (README.md)[https://github.com/cloudframeworks-functionservice/function-example/blob/master/twitter/README.md]

### 实例二：使用FaaS操作ETCD v3

开始前，请确认你已完成以上步骤。若你想要先体验部署应用，可使用本框架维护服务端：http://api.faas.pro

1. 查看源码

   (GitHub地址)[https://github.com/cloudframeworks-functionservice/function-example/tree/master/etcd_v3]

2. 构建与运行应用

   (README.md)[https://github.com/cloudframeworks-functionservice/function-example/blob/master/etcd_v3/README.md]

### Faas应用开发说明

* FaaS应用可使用任何语言进行开发。
* 应用具有一定的运行时间，即完成计算后退出。
* 应用从标准输入或环境变量获取输入数据，以标准输出输出计算结果。
* 应用需要以dockerfile进行镜像打包。

# 框架说明

## 业务

### 业务背景

​      Serverless/FaaS 是一种新的计算范例，可为开发人员和运营商提供简单，高效和可扩展性。对于两者的优势不一样：

1. 对于开发者

* 不需要管理服务器，只需要编辑上传业务代码。其他工作由平台完成。
* 代码微小化，只完成一个功能。维护升级非常简单。
* 需要时执行计算任务，无需支付闲时费用。真正的按需计算和付费。

 若你自己运行本框架，费用问题将不是你的考虑范围。

2. 对于平台运营者

* 资源利用率极高，只有在实际计算时消耗资源。
* 适用于任何语言，任何技术设计的方法的统一运行平台。

## 组件

### 架构图

![](architecture.png)



### Traefik

 了解学习现代化反向代理／负载均衡Traefik:https://traefik.io/

### FunctionAPI

提供一个无状态的API服务。提供应用创建，配置，运行等API。

### Mysql

存储应用方法元数据。

### Redis

消息队列，每次方法调用API产生的任务送往消息队列，调度与执行器从消息队列获取任务并执行。

### Fn

命令行客户端，使你本地开发，本地部署。

### Hub

存储你的方法镜像，使用docker官方镜像仓库服务。项目地址：https://github.com/docker/distribution

# <a name="生产环境"></a>生产环境

`TODO`

# <a name="常见问题"></a>常见问题

`TODO`

# <a name="更新计划"></a>更新计划

`TODO`

点击查看[历史更新](CHANGELOG.md)

# <a name="社群贡献"></a>社群贡献

+ QQ群: 
+ [参与贡献](CONTRIBUTING.md)
+ [联系我们](mailto:info@goodrain.com)

-------

[云框架](ABOUT.md)系列主题，遵循[APACHE LICENSE 2.0](LICENSE.md)协议发布。
