title: Spring Cloud(一)
date: 2018-03-16 23:04:50
tags:
---

## 服务注册发现
在传统的方式中，应用都被部署到IP和Port固定的机器上，通常在配置文件中写死了第三方服务的IP地址和端口。
但是在云环境中，服务器的IP是随机分配的，服务器的数量也是动态变化的， 那么就需要一种服务发现机制, 所以对于微服务治理而言，服务发现和注册是核心。

<!-- more -->

### 类型
#### 客户端发现方式
服务提供者在应用启动或者地址变更的时候会向服务中心注册自己，在应用关闭的时候会向服务中心注销服务，或者应用挂掉了无法发送心跳，服务中心也会撤销服务。
使用客户端的发现方式时，客户端通过查询服务注册中心，得到所有可用的服务的IP地址和端口，然后通过负载均衡算法来选择某个服务。
![client service registry](/uploads/client_service_registry.png)

#### 服务端发现方式
使用服务端的发现方式时，客户端发送请求到负载均衡器，负载均衡器去找到可用的服务，然后转发到某个服务上。
![server service registry](/uploads/server_service_registry.png)

客户端发现模式的优势是，客户端向服务提供者发起请求时比服务端发现模式少了一次网络跳转，劣势是服务消费者需要内置特定的服务发现客户端和服务发现逻辑，针对不同的语言，每个服务的客户端都得实现一套服务发现的功能；
服务端发现模式的优势是服务消费者无需内置特定的服务发现客户端和服务发现逻辑，劣势是多了一次网络跳转，并且需要基础设施环境提供中央路由机制或者负载均衡机制。
目前客户端发现模式应用的多一些，因为这种模式的对基础设施环境没有特殊的要求，和基础设施环境也没有过多的耦合性。

### 服务发现产品
- ZooKeeper是历史最悠久的项目之一，数据存储的格式类似于文件系统，如果运行在一个服务器集群中，Zookeper将跨所有节点共享配置状态，每个集群选举一个领袖，客户端可以连接到任何一台服务器获取数据。
- etcd 高可用，分布式，强一致性的，key-value，Kubernetes和Cloud Foundry都是使用了etcd。
- consul 一个用于discovering和 configuring的工具。它提供了允许客户端注册和发现服务的API。Consul可以进行服务健康检查，以确定服务的可用性。

由于分布式领域的CAP原理，C: 数据一致性，A: 服务可用性，P: 服务对网络分区故障的容错性, 这3个特性在分布式系统中只能满足2个。
而ZooKeeper，etcd等这类产品，它保证的是CP，即不能保证服务的可用性。
对于大部分分布式系统来说，特别是数据存储的系统，数据一致性是首先要保证的，但是对于服务发现，AP胜过CP. 而Spring Cloud Eureka就是保证的AP。

### Eureka

![eureka](/uploads/eureka.png)

#### Eureka Server
即服务的注册中心，通过Spring Cloud来启动一个Eureka server也非常简单。
1. 导入eureka server依赖:
```
dependencies {
    compile 'org.springframework.cloud:spring-cloud-starter-eureka-server'
}
```
2. 加上`@EnableEurekaServer`注解
```
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class);
    }
}
```
3. 加入配置
```
server:
  port: 8761

spring:
  application:
    name: eureka-server //服务注册的name
```
启动后，http://localhost:8761 可以看到eureka ui。

#### Eureka client
服务提供者和服务消费者都称为Eureka client，服务提供者需要注册自己到Server端，服务消费者也需要去Server端获取可用的服务信息。启动方式也非常简单。
1. 导入依赖
```
dependencies {
    compile('org.springframework.cloud:spring-cloud-starter-eureka')
}
```
2. 启动类加`@EnableDiscoveryClient`注解
```
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaClientApplication.class);
    }
}
```
3. 加入配置
```
server:
  port: 8762

spring:
  application:
    name: eureka-client

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka
```
然后，便可以在客户端使用Spring Cloud提供的工具去获取在Server上注册的服务，比如下面代码获取指定service name的信息。
```
@RestController
public class ServiceInstanceController {
    @Autowired
    private DiscoveryClient discoveryClient;

    @GetMapping("client/{client}")
    public List<ServiceInstance> retrieveClientInfo(@PathVariable String client) {
        return discoveryClient.getInstances(client);
    }
}
```
如果之后引入了Spring Cloud Feign等之后，可以非常简单的自动获取服务信息。

#### Eureka HA
Eureka server可以部署集群来解决单点问题。Eureka Server采用的是Peer to Peer对等通信。这是一种去中心化的架构，无master/slave区分，每一个Peer都是对等的。在这种架构中，节点通过彼此互相注册来提高可用性，每个节点需要添加一个或多个有效的serviceUrl指向其他节点。每个节点都可被视为其他节点的副本。

通过如下配置来实现Eureka server间的相互注册。
```
---
spring:
  profiles: peer1                                 # 指定profile=peer1
  application:
    name: Eureka-Server
server:
  port: 8761   # 注册服务的端口号
eureka:
  instance:
    hostname: peer1                               # 指定当profile=peer1时，主机名
  client:
    serviceUrl:
      defaultZone: http://peer2:8762/eureka/      # 将自己注册到peer2这个Eureka上面去

---
spring:
  profiles: peer2
  application:
    name: Eureka-Server
server:
  port: 8762
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: http://peer1:8761/eureka/  # 服务注册地址，将自己注册到peer2上去
```

### 参考文档
[service discovery](https://www.nginx.com/blog/service-discovery-in-a-microservices-architecture/)
[网易Blog](http://tech.lede.com/2017/03/15/rd/server/SpringCloud1/)
[spring cloud eureka](http://blog.didispace.com/spring-cloud-starter-dalston-1/)
