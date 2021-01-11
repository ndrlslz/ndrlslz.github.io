title: Configuration Center
date: 2020-03-11 16:17:31
tags:
---

**Configuration Center**是自己练手写的，一个基于Zookeeper的配置管理中心，包含如下组件。

`configuration-center-api` 提供一个API来创建，浏览，更新，删除配置。
`configuration-center-sdk` 提供一个Java SDK来获取和监听配置，可以在非Spring项目中使用
`spring-boot-starter-configuration-center` 提供了和Spring集合的库，可以在Spring Boot的项目中使用

理论上说这里应该还有个`configuration-center-ui`库来提供UI界面去操作配置项，但我不熟悉前端开发，就没做了。

<!-- more -->

## 快速开始
### 配置数据准备
假设这里有一个`customer-api`应用在`dev`环境。然后我们来创建一些测试用的配置。
理论上来说我们应该通过`configuration-center-api`来创建，但这里为了简单起见，就直接在zookeeper里创建了。

1. 启动zookeeper服务器
2. 在zookeeper服务器上运行下面的命令来创建测试数据
```
create /configuration-center ""
create /configuration-center/customer-api ""
create /configuration-center/customer-api/dev ""
create /configuration-center/customer-api/dev/name "Tom"
```

可以看到，我们创建了为`customer-api`应用在`dev`环境中创建了一个配置`name=Tom`，接下来会介绍如何在非Spring应用或Spring Boot应用中来获取和监听这个配置。

### 非Spring应用
1. 引用`configuration-center-sdk` 库
2. 创建`ConfigurationTemplate`
```
configurationTemplate = new ConfigurationTemplate.Builder()
        .connectionString("localhost:2181")
        .application("customer-api")
        .environment("dev")
        .sessionTimeoutMs(10000)
        .connectionTimeoutMs(10000)
        .build();
```

3. 我们可以通过如下代码来获取配置
`String name = configurationTemplate.get("name")`

4. 也可以监听配置，这样每次`name`配置更新了，回调函数就会被调用。
```
configurationTemplate.listen(this, "name", value -> {
    System.out.println(value);
});
```

### Spring Boot应用
1. 引用`spring-boot-starter-configuration-center`库
2. 在application.yml中添加configuration center相关的所需配置
```
configuration-center:
  application: "customer-api"
  environment: "dev"
  connectionString: "localhost:2181"
```

3. 针对你想从zookeeper中获取的配置，直接在Bean的实例域上加`@Config`注解
```
@RestController
public class TestController {
    @Config(value = "name")
    private String name;

    @RequestMapping("/name")
    public String name() {
        return name;
    }
}
```

4. 当你的配置更改后，如果需要实时获取最新的配置，使用注解`@Config(refresh=true)`
```
@RestController
public class TestController {
    @Config(value = "name", refresh = true)
    private String name;

    @RequestMapping("/name")
    public String name() {
        return name;
    }
}
```

## 设计文档
### 配置结构
zookeeper里的配置结构基本上由4层组成。
最上层是固定的configuration-center的命令空间， 第二层是不同的应用名，第三层是应用所属的不同环境名，最后一层是对应环境的所有配置。

![configuration-center-struct.png](/uploads/configuration-center-struct.png)

### 容灾
configuration center有一套完善的容灾策略，即使configuration center挂掉了也不会影响你应用的启动和配置使用。

1. **本地内存缓存**， 从zookeeper获取的配置会存储在内存中
2. **本地缓存文件**， 应用会持久化内存配置到缓存文件中。当应用无法连接到configuraiton-center并且重启后，因为内存中没有缓存的配置了，这时可以从本地缓存文件中读取配置。
3. **本地容灾文件**， 正常的情况下，容灾文件内是没有内容的。当服务端完全宕机且长时间不能恢复，同时配置又发生了很大的变更时，意味着本地缓存文件的内容已经过时了，可以通过在容灾文件夹内添加文件的方式来开启本地容灾。此时客户端会忽略原有的本地缓存文件，只从本地容灾文件中读取配置
