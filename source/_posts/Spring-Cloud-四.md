title: Spring Cloud(四)
date: 2018-08-28 21:29:53
tags:
---

## Spring Cloud Config
### 简介
在微服务的环境中，有众多的单个服务，每个服务都有自己的一套配置，那么如何更好管理这些配置呢。
我们现在的项目(Spring Boot项目，还没用Spring Cloud)是通过创建一个单独的Configuration的Git仓库，然后在部署的时候，自动化脚本会去Git仓库拉取对应项目的配置，放到config文件夹下，作为一个external的配置。
优势就是集中化管理了所有项目的配置，但是劣势是需要自己在部署脚本中写好拉取配置的代码，以及每次更新了配置，需要重新触发部署才会生效。

Spring Cloud Config是用来集中式的管理分布式项目中的各个项目各个环境的配置。它提供服务端和客户端的支持。
服务端就是一个应用，用来管理所有项目配置，并暴露出接口来访问对应项目，对应profile的配置信息。客户端在启动的时候，通过服务端提供的接口去获取自己的配置信息，

<!-- more -->

### 基本用法
#### 服务端
导入依赖
```
compile('org.springframework.cloud:spring-cloud-config-server')
```

在启动类上加上`@EnableConfigServer`注解
```
@SpringBootApplication
@EnableConfigServer
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class);
    }
}
```

添加配置
```
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/ndrlslz/micro-service
          search-paths: '/config-data/{application}/{profile}'

server:
  port: 1201
```
我的配置长下面这样：
![config_data](/uploads/config_data.png)

启动服务之后，访问配置的URL和配置文件的映射如下：

- /{application}/{profile}[/{label}]
- /{application}-{profile}.yml
- /{label}/{application}-{profile}.yml
- /{application}-{profile}.properties
- /{label}/{application}-{profile}.properties

上面的URL会访问`{application}-{profile}.properties`文件, 上面的label表示git的branch。比如`http://localhost:1201/service-order/docker`可以访问到上图我的配置中的`/config-data/service-order/docker/application.yml`配置。

#### 客户端
导入依赖
```
compile("org.springframework.cloud:spring-cloud-starter-config")
```

在`bootstrap.properties`中配置服务端的信息。
```
spring:
  application:
    name: bff
  cloud:
    config:
      enabled: true
      fail-fast: true //如果无法获取配置，则在启动阶段就失败。
      uri: http://localhost:1201/

management:
  security:
    enabled: false
```
这样bff服务在启动的时候，因为`spring.application.name=bff`, profile默认为`default`， 所以便会去获取`/config-data/bff/default/application.yml`

### 高可用
在生产环境中，由于其他服务高度依赖于配置中心，通常将配置中心部署为高可用的集群。
![config_ha](/uploads/config_ha.png)

让配置中心的集群都指向同一个Git配置仓库，并且将配置中心也注册到Eureka中，这样客户端可以通过客户端负载均衡从其中一个配置中心拉取配置。

将配置信息通过常规做法，引入eureka，将其注册到Eureka Server上，然后客户端通过serviceId去eureka上发现配置中心的服务。
```
spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.serviceId=config-server
```

### 配置刷新
由于这些配置是在客户端启动的时候去拉取的，如果要实现应用在运行过程中刷新，需要引入`actuator`监控模块。其中引入了`/refresh`功能。

在需要支持刷新的变量的类上面加注解`@RefreshScope`。
```
@RestController
@RefreshScope
public class ShopController {
    @Value("${custom_param}")
    private String customParam;
}
```
当配置发生变化的时候，发送POST请求到`/refresh`的endpoint，便可以实现配置刷新。 如果项目简单，可以结合Git的Webhook来实现自动刷新。
但由于微服务的地址是动态改变的，以及微服务往往有很多的单独的服务，所以无法去维护Webhook，这时需要依赖Spring Cloud Bus。

![config_bus](/uploads/config_bus.png)

1、提交代码触发post请求给bus/refresh
2、server端接收到请求并发送给Spring Cloud Bus
3、Spring Cloud bus接到消息并通知给其它客户端
4、其它客户端接收到通知，请求Server端获取最新配置
5、全部客户端均获取到最新的配置

### 参考文档
[配置中心](http://blog.didispace.com/spring-cloud-starter-dalston-3/)
[网易博客](http://tech.lede.com/2017/06/12/rd/server/springCloudConfig/)
[配置刷新](http://www.ityouknow.com/springcloud/2017/05/26/springcloud-config-eureka-bus.html)
