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
         -l traefik.frontend.rule=Host:api.faas.org \
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
            -l traefik.frontend.rule=Host:www.faas.org\
            iron/functions-ui
      ```

   * 安装镜像仓库服务

      ```
      docker run -d --name function-hub \
         --restart always \
         -v `pwd`/data:/var/lib/registry \
         -l traefik.port=5000\
         -l traefik.tags=function-hub\
         -l traefik.frontend.rule=Host:hub.faas.org\
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
             CertFile = "/etc/ssl/faas.org/faas.org.pem"
             KeyFile = "/etc/ssl/faas.org/faas.org.key"

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

      ```
      docker run -d -p 9999:8080 -p 80:80 -p 443:443\
           -v `pwd`/traefik.toml:/etc/traefik/traefik.toml\
           -v /var/run/docker.sock:/var/run/docker.sock\
           -v /etc/ssl/faas.org/:/etc/ssl/faas.org\
           --restart always\
           -v `pwd`/log:/log\
           --name=traefik\
           traefik --web
      ```
