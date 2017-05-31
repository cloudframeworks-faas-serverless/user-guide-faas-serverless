# [云框架]FaaS Serverless架构

[FaaS](http://blog.alexellis.io/functions-as-a-service/)(Function as a Service)／[Serverless](https://martinfowler.com/articles/serverless.html)概念在最初并不为大众所接受，但随着微服务架构及[事件驱动架构](http://microservices.io/patterns/data/event-driven-architecture.html)的发展成熟，越来越多人认识到了其中价值。（[Serverless架构综述](https://martinfowler.com/articles/serverless.html?from=singlemessage&isappinstalled=0)）

简单来说，FaaS／Serverless是一种新的计算范例，为开发者和运营商提供简单、高效、可扩展的计算方法，我们可以把它看作是比微服务更细粒度的架构模式。FaaS／Serverless并不意味着没有服务器，而是通过将复杂的服务器架构透明化，使开发者专注于业务／任务本身，强调了一种减少计算资源关注、工作粒度从服务器切换到“任务”的思想。（[FaaS、PaaS和无服务器体系结构的优势](http://www.infoq.com/cn/news/2016/06/faas-serverless-architecture)）

对于开发者来说，FaaS／Serverless意味着：

* 无需管理服务器，仅需关注业务代码，其他工作将由平台完成

* 代码微小化，只完成一个功能，维护升级非常简单

* 需要时执行计算任务，无需支付闲时费用（真正的按需计算和付费）

对于平台运营者来说，FaaS／Serverless意味着：

* 资源利用率极高，只在实际计算时消耗资源

* 一个适用于任何语言、任何技术设计的方法的统一运行平台

本篇[云框架](ABOUT.md)将以一个自建FaaS平台 http://www.faas.pro 及两个FaaS方法（function）实例[ETCD_v3](https://github.com/cloudframeworks-functionservice/function-example/tree/master/etcd_v3)、[Twitter Function Image](https://github.com/cloudframeworks-functionservice/function-example/tree/master/twitter)为例介绍FaaS／Serverless及其最佳实践。

适用于以下场景：

* 低频请求场景

* 流量突发场景

# 内容概览

* [在线演示](#在线演示)
* [快速部署](#快速部署)
   * [平台部署](#平台部署)
   * [方法实例](#方法实例)
* [框架说明-平台](#框架说明-平台) 
* [FaaS方法开发](#FaaS方法开发)
* [生产环境](#生产环境)
* [常见问题](#常见问题)
* [更新计划](#更新计划)
* [社群贡献](#参与贡献)

# <a name="在线演示"></a>在线演示

http://www.faas.pro

# <a name="快速部署"></a>快速部署

## <a name="平台部署"></a>平台部署

1. [安装Docker及Docker Compose](https://github.com/cloudframeworks-faas-serverless/user-guide-faas-serverless/blob/master/READMORE/installdocker&dockercompose.md)

2. 准备域名（这里使用`faas.org`域名进行部署安装，若你有自己的域名，可使用自己的域名）
    
* 增加本地域名解析（可能需要使用`sudo su`命令切换到root账户）

```
sudo echo "机器IP www.faas.org" >> /etc/hosts
sudo echo "机器IP api.faas.org" >> /etc/hosts
sudo echo "机器IP hub.faas.org" >> /etc/hosts
```

* 获取SSL证书（此处使用faas.org域名，你可以 [构建自签证书并使用自己的域名](https://xiaoai.me/?p=82)）

```
sudo wget -P /etc/ssl/faas.org/ fs.faas.pro/faas.org.key
sudo wget -P /etc/ssl/faas.org/ fs.faas.pro/faas.org.crt
```

3. 平台安装

    * docker-compose安装（执行前请确认已完成系统环境准备）

        ```
        sudo curl fs.faas.pro/dc-up |sh
        ```

        [Mac环境下部署可能报错解决方法](https://github.com/cloudframeworks-functionservice/user-guide-faas/blob/master/READMORE/macdeploysolution.md)

    * 或[基于镜像安装平台组件](https://github.com/cloudframeworks-functionservice/user-guide-faas/blob/master/READMORE/container%20install.md)   

    * 镜像仓库注意：此处部署过程使用自签的ssl证书，使用仓库时需要注意添加自签CA证书或者域名的信任，参考：[搭建安全的Docker Private Registry完全指南](http://dockone.io/article/1277)   

4. 安装Fn客户端([Fn - CLI tool](https://github.com/iron-io/functions/blob/master/fn/README.md)）

    ```
    curl http://fs.faas.pro/fn | sh
    export API_URL=http://api.faas.org
    fn --help
    ```

5. 访问平台

* 通过 http://www.faas.org:9999 查看服务：

<div align=center><img width="900" height="" src="./image/service.png"/></div>

* 通过 http://www.faas.org 进入控制台：

<div align=center><img width="900" height="" src="./image/ui.png"/></div>

# <a name="操作实例"></a>方法实例

**在进行操作实例前需注意：**

* 确保上文平台及Fn客户端已完整部署

* 确认配置环境变量`API_URL=http://api.faas.org`

## ETCD v3 FaaS方法实例

[ETCD_v3介绍](http://www.infoq.com/cn/articles/etcd-interpretation-application-scenario-implement-principle)

[查看方法源码](https://github.com/cloudframeworks-functionservice/function-example/tree/master/etcd_v3)

1. 部署一个ETCD_v3应用 ，若已部署忽略本步骤。 ([部署方式](https://github.com/cloudframeworks-functionservice/function-example/blob/master/etcd_v3/etcd_v3_server.md))

2. 创建应用（etcd_v3 server地址替换`***`）

```
fn apps create --config ETCD_SERVER=*** etcd_v3
```

3. 创建路由

```
fn routes create etcd_v3 /command -i hub.faas.pro/etcd_v3:0.0.1
```

4. 运行方法

* PUT

```
echo '{"method":"put","key":"hello","value":"world"}' | fn call etcd_v3  /command
```

* GET

```
echo '{"method":"get","key":"hello"}' | fn call etcd_v3 /command
```

## Twitter Function Image方法实例

Twitter Function Image可用于查看推文（**需要科学上网**）

[查看方法源码](https://github.com/cloudframeworks-functionservice/function-example/tree/master/twitter)

1. 配置[Twitter App](https://apps.twitter.com/)及[configure Customer Access and Access Token](https://dev.twitter.com/oauth/overview/application-owner-access-tokens)

2. 创建应用(根据你的twitter账号信息更改`***`)

```
fn apps create --config ACCESS_SECRET=*** --config ACCESS_TOKEN=*** --config CUSTOMER_KEY=*** --config CUSTOMER_SECRET=*** twitter
```

3. 创建路由

```
fn routes create twitter /tweets -i hub.faas.pro/func-twitter:0.0.1

```

4. 运行方法,可以使用任何人的账号名替换`***`(username值)

```
echo '{"username":"***"}' | fn call twitter /tweets
```

# <a name="框架说明-平台"></a>框架说明-平台

平台架构图如下所示：

<div align=center><img width="900" height="" src="./image/architecture.png"/></div>

* [Traefik](https://traefik.io/)：了解学习现代化反向代理／负载均衡

* FunctionAPI：提供一个无状态的API服务。提供FaaS方法创建，配置，运行等API

* Mysql：存储FaaS方法元数据

* Redis：消息队列，每次方法调用API产生的任务送往消息队列，调度与执行器从消息队列获取任务并执行

* Fn：命令行客户端，使你本地开发、本地部署

* Hub：存储你的方法镜像，使用docker官方镜像仓库服务。项目地址：https://github.com/docker/distribution

# <a name="开发你的FaaS方法"></a>开发你的FaaS方法

如果想要自己开发FaaS方法，可参考[ETCD_v3](https://github.com/cloudframeworks-functionservice/function-example/tree/master/etcd_v3)、[Twitter Function Image](https://github.com/cloudframeworks-functionservice/function-example/tree/master/twitter)这两个例子。

需要注意的是，与普通应用相比，FaaS方法有以下特点／不同：

* FaaS方法可使用任何语言进行开发
* FaaS方法具有一定的运行时间，即完成计算后退出
* FaaS方法从标准输入或环境变量获取输入数据，以标准输出输出计算结果
* FaaS方法需要以dockerfile进行镜像打包

**步骤：**

1. 使用fn命令构建你的代码：

```
# 创建func.yaml文件 - 用Docker Hub username替换$USERNAME
fn init $USERNAME/hello

# 创建function
fn build

# 测试 - 通过管道传入数据, 例如:`cat hello.payload.json | fn run`
fn run

# 完成后构建并将其push至Docker Hub
fn build && fn push

# 创建一个app - 每个app仅执行一次
fn apps create myapp

# 为function创建一个route
fn routes create myapp /hello
```

2. 使用lambda函数(确保你的node.js主处理函数文件名为：`func.js`)

```
fn init --runtime lambda-node hub.faas.org/lambda-node
```

# <a name="生产环境"></a>生产环境

`TODO`

# <a name="常见问题"></a>常见问题

`TODO`

# <a name="更新计划"></a>更新计划

`TODO`

点击查看[历史更新](CHANGELOG.md)

# <a name="社群贡献"></a>社群贡献

+ QQ群: 457506070
+ [参与贡献](CONTRIBUTING.md)
+ [联系我们](mailto:info@goodrain.com)

-------

[云框架](ABOUT.md)系列主题，遵循[APACHE LICENSE 2.0](LICENSE.md)协议发布。
