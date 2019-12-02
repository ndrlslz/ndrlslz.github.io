title: Spring Cloud(二)
date: 2018-05-05 19:44:27
tags:
---

## Spring Cloud Ribbon
通常为了应用的高可用，我们会部署同一个应用到多台机器上，那客户端在访问这个应用的时候就需要选择访问哪一个实例。负载均衡器的目的就是如此。

<!-- more -->

---
### 负载均衡类型

#### 服务端负载均衡
![server side loadbalancer](/uploads/server_side_loadbalancer.png)
大部分的负载均衡器都是服务端的负载均衡，它可以通过硬件或者软件来实现。
当客户端发送请求到负载均衡器，负载均衡通过某种算法，比如Round-robin（轮询）来决定发给具体某个实例。
常见的服务端的负载均衡比如有基于硬件的F5，基于软件的AWS ELB。

#### 客户端负载均衡
![client side loadbalancer](/uploads/client_side_loadbalancer.png)
随着SOA和微服务的兴起，客户端负载均衡也渐渐变的流行起来，它不需要依赖第三方的负载均衡服务来分发请求，客户端自己就负责去选择发送请求到哪个服务器，

---
### Ribbon基本用法
Ribbon是一个客户端的负载均衡器，使用它最简单的方式就是配置服务发现来用，假设我们已经配置好了Eureka，已经存在一个say-hello的服务，只需要下面简单的代码即可实现负载均衡。
```
@SpringBootApplication
@EnableDiscoveryClient
public class RibbonExampleApplication {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/hi")
    public String hi(@RequestParam String name) {
        String greeting = this.restTemplate.getForObject("http://say-hello/greeting", String.class);
        return String.format("%s, %s!", greeting, name);
    }

    public static void main(String[] args) {
        SpringApplication.run(RibbonExampleApplication.class, args);
    }
}

```


如果单独使用Ribbon的话，不配合服务发现的话,使用的时候，需要`@RibbonClient注解`
```
@SpringBootApplication
@RestController
@RibbonClient(name = "say-hello", configuration = SayHelloConfiguration.class)
public class UserApplication {
...
}
```
并且需要一些额外的配置。比如下面的配置，需要指定say-hello服务在哪些服务器上。
```
say-hello:
  ribbon:
    eureka:
      enabled: false
    listOfServers: localhost:8090,localhost:9092,localhost:9999
    ServerListRefreshInterval: 15000
```
通过上面的配置，会去配置Ribbon client。常见的设置如下：

- ServerList 负载均衡使用的服务器列表, 即上面的listOfServers, 当Ribbon与Eureka结合使用时，ServerList的实现类就是DiscoveryEnabledNIWSServerList，它会保存Eureka Server中注册的服务实例表。
- IPing 探测服务实例是否存活的策略,当Ribbon与Eureka联合使用时，NIWSDiscoveryPing会取代IPing，它将职责委托给Eureka来确定服务端是否已经启动。
- IRule 负载均衡策略，比如轮询，基于相应时间加权，随机等等。
- ILoadBalancer 负载均衡器。这也是一个接口，Ribbon为其提供了多个实现，比如ZoneAwareLoadBalancer。

---
## Spring Cloud Feign
### 概念
Feign是一套基于Netflix Feign实现的声明式服务调用客户端。目的是为了方便的编写Web服务客户端。它和我们之间接触到的Retrofit差不多。我们只需要编写接口代码，用注解来配置它，接下来的事情就都由动态代理帮忙做了。

Feign具有如下特性：

- 可插拔的注解支持，包括Feign注解和JAX-RS注解
- 支持可插拔的HTTP编码器和解码器
- 支持Hystrix和它的Fallback
- 支持Ribbon的负载均衡
- 支持HTTP请求和响应的压缩

### 基本用法
添加`@EnableFeignClients`注解来启用Feign
```
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients //启用Feign
public class Application
{
    public static void main(String[] args)
    {
        SpringApplication.run(Application.class, args);
    }
}
```

创建Feign客户端的接口定义，`@FeignClient`表明调用的服务名称，接口中的方法表明调用的Endpoint。
```
@FeignClient("say-hello")
public interface HelloClient {

    @GetMapping("/greetings")
    String greetings();

}
```

然后就像使用方法一样来使用定义好的接口，正如我们上面提到的，具体的实现都由动态代理帮忙做了。
```
@RestController
public class DcController {

    @Autowired
    HelloClient helloClient;

    @GetMapping("/hello")
    public String hello() {
        return helloClient.greetings();
    }

}
```

### 其他特性
#### 范型支持
```
public interface CrudClient<T> {
    @RequestMapping(method = RequestMethod.POST, value = "/")
    long save(T entity);

    @RequestMapping(method = RequestMethod.PUT, value = "/{id}")
    void update(@PathVariable("id") long id, T entity);

    @RequestMapping(method = RequestMethod.GET, value = "/{id}")
    T get(@PathVariable("id") long id);

    @RequestMapping(method = RequestMethod.DELETE, value = "/{id}")
    void delete(@PathVariable("id") long id);
}
```

#### 异常处理
默认Feign会抛出`FeignException`，它会填充异常信息到序列化的Json字符串，就像下面这样。
```
“feign.FeignException: status 500 reading AccountClient#getAccount(); content:

{“timestamp”:1439806103425,”status”:500,”error”:”Internal Server Error”,”exception”:”java.lang.RuntimeException”,”message”:”Request processing failed; nested exception is java.lang.RuntimeException”,”path”:”/account”}”
```
如果你想要自定义异常信息，参考[ErrorDocoder](https://github.com/OpenFeign/feign/wiki/Custom-error-handling)


#### 异步支持
如果Hystrix在classpath中并且`feign.hystrix.enabled=true`，Feign会默认将所有方法都封装到断路器中。这样一来你就可以使用Reactive Pattern了。(调用.toObservalbe()或.observe()方法，或者通过.queue()进行异步调用)。

将feign.hystrix.enabled=false参数设为false可以关闭对Hystrix的支持。

---
## 参考文档
[feign official document](https://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-feign.html)
[spring cloud feign](http://blog.didispace.com/spring-cloud-starter-dalston-2-3/)
