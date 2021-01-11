title: Spring Boot Plugin带来的Gradle依赖问题
date: 2019-06-09 21:26:26
tags:
---

最近遇到了一个Gradle的依赖问题，但发现使用常见的解决依赖冲突的方式无效，最后发现是因为引入Spring Boot Plugin的问题，在这记录下过程。

<!-- more -->

## 问题描述
我有一个`core`项目，它显式的引入了`guava:23.0`版本。
有另外一个`api`项目，它引入了`core`作为依赖。 表现的问题是`api`项目最终使用的`guava`版本是`18.0`, 而不是期望的`23.0`。

## 尝试使用常用的方式解决依赖问题
一开始很容易想到是依赖冲突的问题，即`api`项目引入了其他的依赖中包含了`guava:18.0`版本，最终覆盖了`core`项目的`guava:23.0`。但其实这个想法不对，因为Gradle默认的依赖管理方式是使用新版本。先不管了，先试试这种办法。

先运行命令看看还有谁在用guava。
```
./gradlew dependencyInsight --configuration compile --dependency guava
```

![gradle_dependency_guava](/uploads/gradle_dependency_guava.png)

发现是`springfox-swagger2`在使用`guava:18.0`版本。 于是将swagger的guava依赖exclude掉。
```
compile('io.springfox:springfox-swagger2:2.7.0') {
    exclude group: 'com.google.guava', module: 'guava'
}
```

但是这种方法无效，`api`项目依然使用`guava:18.0`版本。

## Spring Boot Plugin引发的问题
经过调查，最终是由于引入了Spring Boot的插件所引起的。
```
apply plugin: 'org.springframework.boot'

dependencyManagement {
    imports {
        mavenBom 'org.springframework.cloud:spring-cloud-dependencies:Edgware.RELEASE'
    }
}
```

**Spring Boot的插件有它自己的一套依赖管理规则，如果没有显示地指定依赖版本，它会使用它自定义的依赖规则。**

再回去看下上面的依赖图，其中有一行是
```
com.google.guava:guava:18.0 (selected by rule)
```

`selected by rule`便是一个提示，表示有程序明确地表示使用这个版本。在这个问题里，是Spring Boot的插件自己的Rule明确表示使用`guava:18.0`。

而之前提到过当发生依赖冲突时，Gradle会默认选择新版本来使用。它的表现为`conflict resolution`，类似于下面这样。
```
com.google.guava:guava:23.0 (conflict resolution)
```
所以这个问题并不是依赖冲突的问题，而是Spring Boot的插件带来的问题。

## 解决方案
有多种方式可以解决这个问题，细节可以参考[spring boot plugin官方文档](https://github.com/spring-gradle-plugins/dependency-management-plugin)

我这里是在Spring Boot插件的依赖管理中，显示地声明我希望使用的guava版本。
```
dependencyManagement {
    imports {
        mavenBom 'org.springframework.cloud:spring-cloud-dependencies:Edgware.RELEASE'
    }

    dependencies {
        dependency('com.google.guava:guava:23.0')
    }
}
```

## 参考文档
[Official Documentation](https://docs.spring.io/dependency-management-plugin/docs/current-SNAPSHOT/reference/html5/)
[Gradle Forums](https://discuss.gradle.org/t/excluded-dependence-comes-back-when-spring-boot-plugin-is-applied/17945/2)
[Github Issue](https://github.com/spring-gradle-plugins/dependency-management-plugin/issues/165)
