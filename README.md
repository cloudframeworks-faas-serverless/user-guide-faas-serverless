# [云框架]FaaS Serverless架构

[FaaS](http://blog.alexellis.io/functions-as-a-service/)(Function as a Service)／[Serverless](https://martinfowler.com/articles/serverless.html)概念在最初并不为大众所接受，但随着微服务架构及[事件驱动架构](http://microservices.io/patterns/data/event-driven-architecture.html)的发展成熟，越来越多人认识到了其中价值。（[Serverless架构综述](https://martinfowler.com/articles/serverless.html?from=singlemessage&isappinstalled=0)）

简单来说，FaaS／Serverless是一种新的计算范例，为开发者和运营商提供简单、高效、可扩展的计算方法，我们可以把它看作是比微服务更细粒度的架构模式。FaaS／Serverless并不意味着没有服务器，而是通过将复杂的服务器架构透明化，使开发者专注于业务／任务本身，强调了一种减少计算资源关注、工作粒度从服务器切换到“任务”的思想。

对于开发者来说，FaaS／Serverless意味着：

* 无需管理服务器，仅需关注业务代码，其他工作将由平台完成

* 代码微小化，只完成一个功能，维护升级非常简单

* 需要时执行计算任务，无需支付闲时费用（真正的按需计算和付费）

对于平台运营者来说，FaaS／Serverless意味着：

* 资源利用率极高，只在实际计算时消耗资源

* 一个适用于任何语言、任何技术设计的方法的统一运行平台

本篇[云框架](ABOUT.md)将以一个自建FaaS平台 http://www.faas.pro 及两个FaaS操作实例[ETCD_v3](https://github.com/cloudframeworks-functionservice/function-example/tree/master/etcd_v3)、[Twitter Function Image](https://github.com/cloudframeworks-functionservice/function-example/tree/master/twitter)为例介绍FaaS／Serverless及其最佳实践。

# 内容概览

* [在线演示](#在线演示)
* [快速部署](#快速部署)
   * [平台部署](#平台部署)
   * [操作实例](#操作实例)
* [框架说明-平台](#框架说明-平台) 
* [框架说明-操作实例](#框架说明-操作实例)
* [FaaS应用开发](#FaaS应用开发)
* [生产环境](#生产环境)
* [常见问题](#常见问题)
* [更新计划](#更新计划)
* [社群贡献](#参与贡献)

# <a name="在线演示"></a>在线演示

http://www.faas.pro

# <a name="快速部署"></a>快速部署 （@Barnett 需要把代码解释清楚，并增加一键部署方式）

## <a name="平台部署"></a>平台部署

1. [准备Docker环境](https://github.com/cloudframeworks-functionservice/user-guide-faas/blob/master/READMORE/install%20docker.md)

2. 基于镜像安装组件

   这里我们使用faas.pro域名为例进行说明。你需要使用你的域名更换类似`traefik.frontend.rule=Host:api.faas.pro`中的域名指定。

   * 安装数据持久化服务MYSQL

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

      上述步骤完成后访问`<你的域名或IP>:9999` 你将看到下图所示服务：

      ![](./image/service.png)

      访问`<你的域名或IP>`可以进入控制台：

      ![](./image/ui.png)

3. 安装Fn客户端

   ```
   curl http://fs.faas.pro/fn | sh
   export API_URL=http://api.faas.pro
   fn --help
   ```

## <a name="操作实例"></a>操作实例

**注意：进行以下步骤首先需确保上文平台及Fn客户端已完整部署**

### ETCD v3 FaaS操作实例

1. ETCD v3 server ([好雨部署](etcd_v3_server.md))

2. Development

   2.1 构建本地镜像

      ```
      # 修改func.yaml文件，将name改成你自己的镜像名称。
      # build it

      fn build
      ```

   2.2 本地测试

      ```
      fn run
      ```

   2.3 部署应用到仓库

      ```
      fn deploy etcd_v3
      ```

3. 在平台运行

   3.1 首先设置必须的环境变量

      ```
      # Set your Function server address
      # Eg. api.faas.pro

      FUNCAPI=YOUR_FUNCTIONS_ADDRESS

      # ETCD服务端地址需要先部署etcd,参考： (Requirements)[#Requirements]

      ETCD_SERVER=""
      ```

   3.2 Running with Functions

   * 创建应用

      ```
      curl -X POST --data '{
          "app": {
              "name": "etcd_v3",
              "config": { 
                  "ETCD_SERVER": "'$ETCD_SERVER'",
              }
          }
      }' http://$FUNCAPI/v1/apps
      ```

   * 创建路由

      ```
       curl -X POST --data '{
          "route": {
              "image": "<镜像名>",
              "path": "/command",
          }
      }' http://$FUNCAPI/v1/apps/etcd_v3/routes
      ```

4. 云端运行

   ```
   curl -X POST --data '{"method": "put","key":"/hello","value":"hello word"}' http://$FUNCAPI/r/etcd_v3/command
   curl -X POST --data '{"method": "get","key":"/hello"}' http://$FUNCAPI/r/etcd_v3/command

   ```

### Twitter Function Image操作实例

1. Configure a [Twitter App](https://apps.twitter.com/) and [configure Customer Access and Access Token](https://dev.twitter.com/oauth/overview/application-owner-access-tokens).

2. Development

   2.1 构建本地镜像

      ```
      # 修改func.yaml文件，将name改成你自己的镜像名称。
      # build it

      fn build
      ```
   
   2.2 本地测试

      ```
      fn run
      ```

   2.3 上传到镜像仓库

      ```
      docker push <镜像名>
      ```

3. 在平台运行

   3.1 首先设置必须的环境变量

      ```
      # Set your Function server address
      # Eg. api.faas.pro
   
      FUNCAPI=YOUR_FUNCTIONS_ADDRESS
   
      # 以下信息在 apps.twitter.com 申请和获取 (Requirements)[#Requirements]
      CUSTOMER_KEY="XXXXXX"
      CUSTOMER_SECRET="XXXXXX"
      ACCESS_TOKEN="XXXXXX"
      ACCESS_SECRET="XXXXXX"
      ```

   3.2 Running with Functions

   * 创建应用

      ```
      curl -X POST --data '{
          "app": {
              "name": "twitter",
              "config": { 
                  "CUSTOMER_KEY": "'$CUSTOMER_KEY'",
                  "CUSTOMER_SECRET": "'$CUSTOMER_SECRET'", 
                  "ACCESS_TOKEN": "'$ACCESS_TOKEN'",
                  "ACCESS_SECRET": "'$ACCESS_SECRET'"
              }
          }
      }' http://$FUNCAPI/v1/apps
      ```

   * 创建路由

      ```
      curl -X POST --data '{
          "route": {
              "image": "<镜像名>",
              "path": "/tweets",
          }
      }' http://$FUNCAPI/v1/apps/twitter/routes
      ```

4. 云端运行

   ```
   curl -X POST --data '{"username": "想要获取的Twitter账户名"}' http://$FUNCAPI/r/twitter/tweets
   ```

## <a name="框架说明-业务"></a>框架说明-平台

（添加平台说明文字 @YEYU）

平台架构图如下所示：

![](./image/architecture.png)

* Traefik：了解学习现代化反向代理／负载均衡（Traefik:https://traefik.io/）

* FunctionAPI：提供一个无状态的API服务。提供应用创建，配置，运行等API

* Mysql：存储应用方法元数据

* Redis：消息队列，每次方法调用API产生的任务送往消息队列，调度与执行器从消息队列获取任务并执行

* Fn：命令行客户端，使你本地开发、本地部署

* Hub：存储你的方法镜像，使用docker官方镜像仓库服务。项目地址：https://github.com/docker/distribution

## <a name="框架说明-应用"></a>框架说明-操作实例 （@Barnett 添加实际操作演示）

ETCD v3 FaaS操作实例

Twitter Function Image操作实例

## <a name="FaaS应用开发"></a>FaaS应用开发

如果想要自己开发FaaS应用，可参考[ETCD_v3](https://github.com/cloudframeworks-functionservice/function-example/tree/master/etcd_v3)、[Twitter Function Image](https://github.com/cloudframeworks-functionservice/function-example/tree/master/twitter)这两个例子。

需要注意的是，与普通应用相比，FaaS应用有以下特点／不同：

* FaaS应用可使用任何语言进行开发

* 应用具有一定的运行时间，即完成计算后退出

* 应用从标准输入或环境变量获取输入数据，以标准输出输出计算结果

* 应用需要以dockerfile进行镜像打包

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
