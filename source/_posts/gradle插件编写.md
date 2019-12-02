title: gradle插件编写
date: 2017-11-25 21:19:01
tags:
---

最近需要开发一个控制TrafficManager(一种应用层负载均衡产品)的工具，原理非常简单，就只需要通过输入来调用TrafficManager提供的API而已。
只不过需要已Gradle插件的形式来写这个工具，所以也学习了下怎么样写一个Gradle Plugin。

---

<!-- more -->
### 编写插件
1. 新建HelloPlugin的文件夹，进去后初始化一个Groovy的项目： `gradle init --type=groovy-library`
2. 修改`build.gradle`文件
```
apply plugin: 'idea'
apply plugin: 'groovy'
apply plugin: 'maven'

dependencies {
    compile gradleApi()
    compile localGroovy()
}

repositories {
    mavenCentral()
}

group='com.test'
version='1.0.0'

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: uri('../repo'))
        }
    }
}
```

3. 通过继承Plugin<T>接口可以定义我们自己的插件,当我们这个插件被具体Project用到的时候，Gradle会实例化这个HelloPlugin类，然后会调用apply(T)方法，把Project作为参数穿进去。
这样的话我们可以在这个插件里面配置这个Project。包括创建task等。
```
class HelloPlugin implements Plugin<Project>{
    private final static String PLUGIN_NAME = "hello"

    @Override
    void apply(Project project) {
        project.extensions.create(PLUGIN_NAME, HelloExtension)
        project.tasks.create(PLUGIN_NAME, HelloTask)
    }
}
```

4. 定义Extension。通常的插件都需要从外部接收配置参数，Gradle的每个Project都有一个ExtensionContainer,用来保存所有的配置参数。
我们可以通过添加自己的Extension到ExtensionContainer中，这样就可以从外部接收参数了。通过上一步可以看到我们把HelloExtension加到Project中了。
```
class HelloExtension {
    String message = "hello"
}
```

5. 定义Task,通过继承DefaultTask
```
class HelloTask extends DefaultTask {
    HelloExtension extension;

    @TaskAction
    def hello() {
        extension = project.extensions.findByType(HelloExtension.class)
        println extension.message
    }
}
```

6. 定义插件的ID
当其他项目要用到我们的插件的时候，通过`apply plugin: 'your-plugin-id'`即可，这里就是使用的插件的ID。
创建文件`src/main/resources/META-INF/gradle-plugins/hello-plugin.properties`，插件ID即为文件的名字(hello-plugin)，然后内容为
`implementation-class=com.test.HelloPlugin`,实现类指向我们的插件类。

7. 发布插件
如果想要发布到公共库，参考[发布到公共环境](https://plugins.gradle.org/docs/submit)
但我们这只发布到本地的文件系统即可，通过第2步可以看到。

---
### 使用插件
当我们将插件发布到本地文件系统后，新创建一个项目来使用它，在新项目的`build.gradle`中添加下面代码
```
buildscript {
    repositories {
        maven {
            url uri('../repo')
        }
    }
    dependencies {
        classpath group: 'com.test', name: 'HelloPlugin',
                  version: '1.0.0'
    }
}
apply plugin: 'hello-plugin'

hello {
  message = "hello plugin!"
}
```

然后便可以使用我们插件的中hello方法了`gradlew hello`

---
### 参考文档
[官方文档](https://docs.gradle.org/current/userguide/custom_plugins.html)
