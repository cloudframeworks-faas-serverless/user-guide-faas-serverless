## 安装 docker

#### centos

   ```
   1.清除docker 旧版本
    
     rpm -qa |grep docker
     yum  -y  remove docker* 
        
   2.安装新的docker
    
     yum install -y docker-engine
        
   3.systemctl  start docker
    
   4.docker info 查看docker状态
   ```

#### ubuntu

   ```
   1.更新apt包
    
     sudo apt-get update
        
   2.安装 Docker
    
     sudo apt-get install docker-engine
        
   3.sudo service docker start
    
   4.docker info 查看docker状态
   ```

#### mac

   请参考[https://docs.docker.com/docker-for-mac/](https://docs.docker.com/docker-for-mac/)
## 安装docker-compose
```
curl -L https://github.com/docker/compose/releases/download/1.13.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```   
## 准备域名
本文框架使用`faas.org`域名进行部署安装，若你有自己的域名，请使用自己的域名。
1. 增加本地域名解析

```
# 若你的机器IP是192.168.0.100
sudo echo "192.168.0.100 www.faas.org" >> /etc/hosts
sudo echo "192.168.0.100 api.faas.org" >> /etc/hosts
sudo echo "192.168.0.100 hub.faas.org" >> /etc/hosts
```

2. 获取SSL证书
* 使用faas.org域名

```
sudo wget -P /etc/ssl/faas.org/ fs.faas.pro/faas.org.key
sudo wget -P /etc/ssl/faas.org/ fs.faas.pro/faas.org.crt
```

* 使用自己的域名 >> [构建自签证书](https://xiaoai.me/?p=82) / [购买一年免费阿里云域名证书]()

