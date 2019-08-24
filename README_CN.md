
# 如何运行

启动8个spring boot项目, 4个MongoDB实例, 一个RabbitMQ.

## 启动前准备

* 安装 docker 和 docker compose
* 设置环境变量:

```
export CONFIG_SERVICE_PASSWORD=configpassword
export NOTIFICATION_SERVICE_PASSWORD=notificationpassword
export STATISTICS_SERVICE_PASSWORD=statisticspassword
export ACCOUNT_SERVICE_PASSWORD=accountpassword
export MONGODB_PASSWORD=mongodbpassword
```

* 构建项目: mvn package -Dmaven.test.skip=true

## 开发模式

```
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up
```


# 项目架构

## 配置中心 config service

项目 `config` 实现了配置中心的功能, 但是并没有注册到 `Eureka` 中进行管理, 每个  `config client` 直接访问 `config server`来获取所需的配置文件.

`config server` 本身使用本地存储的, 配置文件统一放置待 `classpath:/shared` 下.

`config server` 对外暴露端口8888, `config client` 使用 `http://config:8888` 直接访问, `uri` 中的 `config` 是 `docker` 容器名.

当 `Notification-service` 请求他的配置文件时, `config service` 会返回 `shared/notification-service.yml` 和 `shared/application.yml`. 


```
spring.cloud.config.fail-fast=true  #如果 config client 不能连接上 config server, 则立即终止启动
```

### application.yml

`config server` 服务自身的配置文件

``` 
spring:
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/shared
  profiles:
     active: native
  security:
    user:
      password: ${CONFIG_SERVICE_PASSWORD}

server:
  port: 8888
```

配置解析:

``` 
spring.cloud.config.server.native.search-locations: 指定本地配置文件的搜索目录
spring.profiles.active: 设置成 native, 使用本地存储配置文件的方式
```

### shared/applicaiton.yml

当任何一个 `config client` 来查询配置文件的时候, 除了返回和客户端名字一样的配置文件, 还会返回这个 application.yml.
因为每个 `config client` 中仅仅有一个 `bootstrap.yml` 文件用来访问 `config server`. **TODO: 关于这个特性, 需要查源码确认下实现逻辑**

``` 
logging:
  level:
    org.springframework.security: INFO

hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 10000

eureka:
  instance:
    prefer-ip-address: true
  client:
    serviceUrl:
      defaultZone: http://registry:8761/eureka/

security:
  oauth2:
    resource:
      user-info-uri: http://auth-service:5000/uaa/users/current

spring:
  rabbitmq:
    host: rabbitmq
```

配置解析:

``` 
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 命令执行超时时间
prefer-ip-address: 控制服务注册时, 使用的是 hostname 还是 ip, 默认是false, 使用hostname进行服务注册,一级服务信息的显示
eureka.client.serviceUrl.defaultZone: 指定服务注册中心的地址
```

## 服务注册中心 registry

### bootstrap.yml

因为这个服务是服务注册中心, 所以除了指定配置中心的相关参数外, 还特别配置了一下 `Eureka server` 自由的参数.

``` 
spring:
  application:
    name: registry
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
      password: ${CONFIG_SERVICE_PASSWORD}
      username: user

eureka:
  instance:
    prefer-ip-address: true
  client:
    registerWithEureka: false
    fetchRegistry: false
    server:
      waitTimeInMsWhenSyncEmpty: 0
```

配置解析:

``` 
prefer-ip-address: 控制服务注册时, 使用的是 hostname 还是 ip, 默认是false, 使用hostname进行服务注册,一级服务信息的显示
registerWithEureka: 表示是否注册自身到eureka服务器，因为当前这个应用就是eureka服务器，没必要注册自身，所以这里是false
fetchRegistry: 表示是否从eureka服务器获取注册信息
```

### shared/registry.yml

``` 
server:
  port: 8761
```

作为服务注册中心, 指定自身的服务端口.

## 账户服务 account service

### bootstrap.yml

指定要访问的配置中心

``` 
spring:
  application:
    name: account-service
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
      password: ${CONFIG_SERVICE_PASSWORD}
      username: user
```


### shared/account-service.yml

``` 
security:
  oauth2:
    client:
      clientId: account-service
      clientSecret: ${ACCOUNT_SERVICE_PASSWORD}
      accessTokenUri: http://auth-service:5000/uaa/oauth/token
      grant-type: client_credentials
      scope: server

spring:
  data:
    mongodb:
      host: account-mongodb
      username: user
      password: ${MONGODB_PASSWORD}
      database: piggymetrics
      port: 27017

server:
  servlet:
    context-path: /accounts
  port: 6000

feign:
  hystrix:
    enabled: true
```

配置解析:

* 配置oauth认证
* 配置mogodb访问
* `server.servlet.context-path`: 设置服务的上下文路径, 会成为 url 中的一部分
* `feign.hystrix.enabled`: 开启 hystrix