
# 如何运行

启动8个spring boot项目, 4个MongoDB实例, 一个RabbitMQ.

## 启动前准备

* 安装 docker 和 docker compose
* 设置环境变量:

直接export环境变量, 这种方式仅仅对当前session有效, 所以后续启动也需要在这个session下

```shell
export CONFIG_SERVICE_PASSWORD=configpassword
export NOTIFICATION_SERVICE_PASSWORD=notificationpassword
export STATISTICS_SERVICE_PASSWORD=statisticspassword
export ACCOUNT_SERVICE_PASSWORD=accountpassword
export MONGODB_PASSWORD=mongodbpassword
```

* 构建项目: mvn package -Dmaven.test.skip=true

## 开发模式启动

```shell
docker-compose -f docker-compose.yml -f docker-compose.dev.yml updocker-compose.yml 解析
```


# 项目架构

## pom.xml 解析

`spring-cloud-dependencies`对所有的cloud依赖进行管理, 如：`spring-cloud-starter-config` 、`spring-cloud-starter-netflix-eureka-server` 

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

  <!-- 项目基本信息 -->
	<groupId>com.piggymetrics</groupId>
	<artifactId>piggymetrics</artifactId>
	<version>1.0-SNAPSHOT</version>
	<packaging>pom</packaging>
	<name>piggymetrics</name>

  <!-- 使用spring-boot -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.3.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

  <!-- 常用属性 -->
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
		<java.version>1.8</java.version>
	</properties>

  <!-- spring cloud 依赖管理 -->
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
	
  <!-- 模块清单 -->
	<modules>
		<module>config</module>
		<module>monitoring</module>
		<module>registry</module>
		<module>gateway</module>
		<module>auth-service</module>
		<module>account-service</module>
		<module>statistics-service</module>
		<module>notification-service</module>
		<module>turbine-stream-service</module>
	</modules>

</project>
```



## docker-compose.yml 解析

```yaml
version: '2.1'
services:
  rabbitmq:
    image: rabbitmq:3-management													# rabbitmq镜像不需要手动构建
    restart: always
    ports:
      - 15672:15672
    logging:																							# 配置日志服务
      options:
        max-size: "10m"
        max-file: "10"

  config:																									# 配置中心
    environment:
      CONFIG_SERVICE_PASSWORD: $CONFIG_SERVICE_PASSWORD		# 从环境变量里获取访问配置中心的密码
    image: sqshq/piggymetrics-config											# 手动构建
    restart: always
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  registry:																								# 服务注册中心
    environment:
      CONFIG_SERVICE_PASSWORD: $CONFIG_SERVICE_PASSWORD
    image: sqshq/piggymetrics-registry										# 手动构建
    restart: always
    depends_on:																			# 设置依赖的容器为config，并且检查容器健康状态
      config:
        condition: service_healthy												# version3不在支持condition
    ports:
      - 8761:8761
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  gateway:																								# 网关
    environment:
      CONFIG_SERVICE_PASSWORD: $CONFIG_SERVICE_PASSWORD
    image: sqshq/piggymetrics-gateway											# 手动构建
    restart: always
    depends_on:
      config:
        condition: service_healthy
    ports:
      - 80:4000
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  auth-service:																							# 认证服务
    environment:
      CONFIG_SERVICE_PASSWORD: $CONFIG_SERVICE_PASSWORD
      NOTIFICATION_SERVICE_PASSWORD: $NOTIFICATION_SERVICE_PASSWORD
      STATISTICS_SERVICE_PASSWORD: $STATISTICS_SERVICE_PASSWORD
      ACCOUNT_SERVICE_PASSWORD: $ACCOUNT_SERVICE_PASSWORD
      MONGODB_PASSWORD: $MONGODB_PASSWORD
    image: sqshq/piggymetrics-auth-service									# 手动构建
    restart: always
    depends_on:
      config:
        condition: service_healthy
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  auth-mongodb:																							# 认证服务的数据库
    environment:
      MONGODB_PASSWORD: $MONGODB_PASSWORD
    image: sqshq/piggymetrics-mongodb												# 手动构建
    restart: always
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  account-service:																					# 账户服务
    environment:
      CONFIG_SERVICE_PASSWORD: $CONFIG_SERVICE_PASSWORD
      ACCOUNT_SERVICE_PASSWORD: $ACCOUNT_SERVICE_PASSWORD
      MONGODB_PASSWORD: $MONGODB_PASSWORD
    image: sqshq/piggymetrics-account-service								# 手动构建
    restart: always
    depends_on:
      config:
        condition: service_healthy
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  account-mongodb:																					# 账户服务的数据库
    environment:
      INIT_DUMP: account-service-dump.js
      MONGODB_PASSWORD: $MONGODB_PASSWORD
    image: sqshq/piggymetrics-mongodb												# 手动构建
    restart: always
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  statistics-service:																				# 计算服务
    environment:
      CONFIG_SERVICE_PASSWORD: $CONFIG_SERVICE_PASSWORD
      MONGODB_PASSWORD: $MONGODB_PASSWORD
      STATISTICS_SERVICE_PASSWORD: $STATISTICS_SERVICE_PASSWORD
    image: sqshq/piggymetrics-statistics-service						# 手动构建
    restart: always
    depends_on:
      config:
        condition: service_healthy
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  statistics-mongodb:																			# 计算服务的数据库
    environment:
      MONGODB_PASSWORD: $MONGODB_PASSWORD
    image: sqshq/piggymetrics-mongodb											# 手动构建
    restart: always
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  notification-service:																		# 通知服务
    environment:
      CONFIG_SERVICE_PASSWORD: $CONFIG_SERVICE_PASSWORD
      MONGODB_PASSWORD: $MONGODB_PASSWORD
      NOTIFICATION_SERVICE_PASSWORD: $NOTIFICATION_SERVICE_PASSWORD
    image: sqshq/piggymetrics-notification-service				# 手动构建
    restart: always
    depends_on:
      config:
        condition: service_healthy
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  notification-mongodb:																		# 通知服务的数据库
    image: sqshq/piggymetrics-mongodb											# 手动构建
    restart: always
    environment:
      MONGODB_PASSWORD: $MONGODB_PASSWORD
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  monitoring:																							# Hystrix Dashboard
    environment:
      CONFIG_SERVICE_PASSWORD: $CONFIG_SERVICE_PASSWORD
    image: sqshq/piggymetrics-monitoring									# 手动构建
    restart: always
    depends_on:
      config:
        condition: service_healthy
    ports:
      - 9000:8080
    logging:
      options:
        max-size: "10m"
        max-file: "10"

  turbine-stream-service:																	# 集群监控
    environment:
      CONFIG_SERVICE_PASSWORD: $CONFIG_SERVICE_PASSWORD
    image: sqshq/piggymetrics-turbine-stream-service			# 手动构建
    restart: always
    depends_on:
      config:
        condition: service_healthy
    ports:
    - 8989:8989
    logging:
      options:
        max-size: "10m"
        max-file: "10"


```



## 配置中心 config service

项目 `config` 实现了配置中心的功能, 但是并没有注册到 `Eureka` 中进行管理, 每个  `config client` 直接访问 `config server`来获取所需的配置文件.

`config server` 本身使用本地存储的, 配置文件统一放置待 `classpath:/shared` 下.

`config server` 对外暴露端口8888, `config client` 使用 `http://config:8888` 直接访问, `uri` 中的 `config` 是 `docker` 容器名.

当 `Notification-service` 请求他的配置文件时, `config service` 会返回 `shared/notification-service.yml` 和 `shared/application.yml`. 


```shell
spring.cloud.config.fail-fast=true  #如果 config client 不能连接上 config server, 则立即终止启动
```

### config/Dockerfile

config 镜像在构建的时候设置了`HEALTHCHECK`, 用来检查容器的健康状况. 从 `docker-compose.yml` 可以看出来, 后续每个容器都依赖 config 容器, 并且都要检查 config 服务的健康状态.

```dockerfile
FROM java:8-jre	# 依赖的镜像
MAINTAINER Alexander Lukyanchikov <sqshq@sqshq.com>

ADD ./target/config.jar /app/	# 添加 jar 包
CMD ["java", "-Xmx200m", "-jar", "/app/config.jar"]	# 容器启动命令

# 设置检查容器健康状况
HEALTHCHECK --interval=30s --timeout=30s CMD curl -f http://localhost:8888/actuator/health || exit 1

EXPOSE 8888
```

### application.yml

`config server` 服务自身的配置文件

``` yaml
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

``` shell
spring.cloud.config.server.native.search-locations: 指定本地配置文件的搜索目录
spring.profiles.active: 设置成 native, 使用本地存储配置文件的方式
```

### shared/applicaiton.yml

当任何一个 `config client` 来查询配置文件的时候, 除了返回和客户端名字一样的配置文件, 还会返回这个 application.yml.
因为每个 `config client` 中仅仅有一个 `bootstrap.yml` 文件用来访问 `config server`. **TODO: 关于这个特性, 需要查源码确认下实现逻辑**

``` yaml
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

``` shell
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 命令执行超时时间
prefer-ip-address: 控制服务注册时, 使用的是 hostname 还是 ip, 默认是false, 使用hostname进行服务注册,一级服务信息的显示
eureka.client.serviceUrl.defaultZone: 指定服务注册中心的地址
```

## 服务注册中心 registry

### registry/Dockerfile

registry没有什么特殊的要求, 直接启动 account-service.jar 就行了.

```dockerfile
FROM java:8-jre
MAINTAINER Alexander Lukyanchikov <sqshq@sqshq.com>

ADD ./target/account-service.jar /app/
CMD ["java", "-Xmx200m", "-jar", "/app/account-service.jar"]

EXPOSE 6000
```



### bootstrap.yml

因为这个服务是服务注册中心, 所以除了指定配置中心的相关参数外, 还特别配置了一下 `Eureka server` 自由的参数.

``` yaml
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

``` shell
prefer-ip-address: 控制服务注册时, 使用的是 hostname 还是 ip, 默认是false, 使用hostname进行服务注册,一级服务信息的显示
registerWithEureka: 表示是否注册自身到eureka服务器，因为当前这个应用就是eureka服务器，没必要注册自身，所以这里是false
fetchRegistry: 表示是否从eureka服务器获取注册信息
```

### shared/registry.yml

``` yaml
server:
  port: 8761
```

作为服务注册中心, 指定自身的服务端口.

## 账户服务 account service

### bootstrap.yml

指定要访问的配置中心

``` yaml
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

``` yaml
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


















































