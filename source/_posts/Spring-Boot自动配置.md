title: Spring Boot自动配置
date: 2017-12-21 21:08:38
tags:
---

相信所有用过Spring Boot的人都同意Spring boot极大的简化了开发的工作，提高了开发的效率。所以就学习了下Spring Boot是如何做到的。

<!-- more -->


---
### Spring boot自动配置
Spring Boot的核心原理就在于自动配置，它遵循`习惯优于配置`的理念，帮我们做了大量的配置的工作。


#### 依赖管理的自动配置
首先是依赖的自动配置，Spring Boot提供的很多的starter，通过引入starter，Spring会帮你引入一系列的依赖，原来的情况是，当我想开发一个功能的时候，我还得思考我应该引入哪些依赖包，引入的版本应该是什么，有时还会遇到版本冲突的情况，而通过spring-boot-dependencies维护了一份庞大的依赖，经过实践，不会产生冲突。
下列是一些starter例子。

| name | description |
| ---  | ---         |
| spring-boot-starter | 核心的starter, 包括了自动配置的库，日志库等|
| spring-boot-starter-web | 包括了web开发的依赖，比如spring mvc等 |
| spring-boot-starter-data-jpa | 对Java持久化API的支持，包括一些spring jdbc，hibernate等等的库 |
| spring-boot-starter-freemarker | 对freemarker模版引擎的支持 |

#### 自动配置Bean
Spring自动配置的代码在`spring-boot-autoconfigure`库中，里面包含的Spring boot为我们做好的配置。这个库被`spring-boot-starter`依赖，而`spring-boot-starter`又被其他starter依赖。那它是怎么为我们自动配置的呢。

我们开发的Spring Boot项目的启动类都有一个注解叫`SpringBootApplication`，它的定义如下，实际就是几个注解的组合。
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
...
}
```
1. `SpringBootConfiguration`注解实际就是`configuration`，就是用JavaConfig的形式来配置Bean
2. `ComponentScan`也很熟悉了，用来指定扫描哪些package来注入Bean，默认会扫描使用`ComponentScan`注解的类所在的package。所以最好把启动类放在root package,这样就不用指定basePackages了。
3. 最重要的就是这个`EnableAutoConfiguration`注解了。
它的定义如下
```
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
...
}
```
`EnableAutoConfiguration`这个注解的实现方式和`EnableCache`,`EnableScheduling`等注解的实现方式是一样的，就是通过`import`来导入一些Configuration,只不过这里导入的是个Selector,它会有选择的把满足条件的Configuration中的Bean都注入到IoC容器中。

`EnableAutoConfigurationImportSelector`里面做的事情是通过`SpringFactoriesLoader`去加载`META-INF/spring.factories`配置，下面是`spring.factories`的部分内容。
```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
```
可以看到，这个文件里面声明了需要自动加载哪些Configuration到IoC容器里面，当然也不是强制的加载所有的，一般都会有有一些条件，满足这些条件才会加载。

所以自动配置的原理就在于Spring boot会扫描classpath下的所有`META-INF/spring.factories`, 将里面的满足条件的Configuration自动的注入到IoC容器中。

![Spring Boot](/uploads/springboot-autoconfigure.png)

---
### Spring Boot热部署
在开发Spring Boot项目的时候，经常因为改动而重启项目，`spring-boot-devtools`提供了开发时候的热部署功能。在gradle中导入devtools，推荐使用netflix提供的插件，它提供了optional 依赖的功能。
```
compile 'org.springframework.boot:spring-boot-devtools', optional
```
它的工作方式是：Spring Boot会使用2个classloader，base classloader用于加载不变类（第三方jar），另一个（restart classloader）用于加载项目的类。当项目重启的时候，只需要restart classloader重新加载项目代码。它的触发方式是classpath下的文件发生改变。

使用方式：如果使用Eclipse，保存文件就会触发classpath中文件改变，Idea的话需要重新编译下(Build->make project)。



---
### 参考文档
[define-own-starter](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-auto-configuration.html)
[spring-boot-autoconfigure](http://tengj.top/2017/03/09/springboot3/)
