title: SPI实现可插拔组件
date: 2017-01-19 21:19:31
tags:
---

### 概念
**SPI(Service Provider Interface)**, 从字面上理解是给服务提供者的接口。 一个接口可以有多种实现方式，通过SPI机制, 我们可以找到这个接口的某个或者所有的实现方式,有点类似与服务发现,通常在写Library或者Framework的时候需要用到。

---

### 基本用法
我们尝试定义1个接口和2个实现类,然后通过SPI机制来通过这个接口找到2个实现类。

<!-- more -->

#### 接口和实现类
```
//interface
interface IService {
    void run();
}

//implemention one
public class HttpService implements IService {
    public void run() {
        System.out.println("Http Service");
    }
}

//implemention two
public class RpcService implements IService{
    public void run() {
        System.out.println("Rpc service");
    }
}
```

#### 定义接口和对应实现的文件
然后需要在resources文件夹下创建`META-INF`文件夹，再下面创建`services`文件夹,然后创建文件，文件名为接口的路径(包含包名)，文件内容为实现类的路径(包含包名)。
在这个例子中，项目结构为：
![spi基本用法图](/uploads/spi-basic-usage.png)

#### 使用方式
通过SPI，我们指定IServcie这个接口，它可以找到我们指定在文件中的实现类。
```
public class Main {
    public static void main(String[] args) {
        Iterator<IService> iterator = ServiceLoader.load(IService.class).iterator();

        IService service;

        while (iterator.hasNext()) {
            service = iterator.next();
            service.run();
        }
    }
}

//Http Service
//Rpc service
```

---

### 可插拔组件的实现
在上一篇的sdk中用到了Retrofit,其中有一段代码是:
```
return new Retrofit.Builder()
        .baseUrl(baseUrl)
        .addConverterFactory(GsonConverterFactory.create(gson))
        .addCallAdapterFactory(ServiceCallAdapterFactory.create())
        .client(client)
        .build()
        .create(ZhihuDaily.class);
```
在第三行我们可以定义序列化的实现方式，如果我们想把Gson改成Jackson的话就需要改第三行。现在，我们用SPI的机制来提供一个库，当使用者想从Gson改成Jackson的话，不用更改代码，只需更改依赖就可以实现。

现在假设我们要实现下面这个多组件的项目。其中，Core模块是核心模块，它调用了另一个负责序列化的模块，但是另一个模块的实现方式是可选的，可以用Gson或者Jackson。
![spi模块设计图](/uploads/spi-module-design.png)

#### Core模块
先定义一个序列化的接口。
```
package com.test.core;

public interface IService {
    void process();
}
```

然后定义一个提供给用户使用的类，其中这个类使用了SPI，会去找IService接口的所有实现方式，但只选择第一个。
```
package com.test.core;

public class Retrofit {
    public static void run() {
        Iterator<IService> iterator = ServiceLoader.load(IService.class).iterator();
        IService service;
        if (iterator.hasNext()) {
            service = iterator.next();
        } else {
            throw new RuntimeException("No implementation of IService.");
        }
        service.process();
    }
}
```

#### Gson模块
Gson模块会依赖Core模块，因为会实现IService接口。 定义个GsonServcie实现类。
```
package com.test.gson;

public class GsonService implements IService {
    @Override
    public void process() {
        System.out.println("invoke Gson service");
    }
}
```

然后创建接口和实现的文件。
```
//resources/META-INF/services/com.test.core.IService
com.test.gson.GsonService
```

#### Jackson模块
参考Gson模块。

整体项目图：
![spi模块实现图](/uploads/spi-module-implemention.png)

#### 使用方式
然后当用户调用Core模块中的Retrofit的时候，通过改变依赖来改变使用的序列化方式。Core模块一定要导入，然后Gson和Jackson模块导入其中一个即可，导入哪个，Core模块就会自动使用那个模块，切换的话也只需要改变依赖即可。
如果用户对使用哪个序列化库无所谓的话，就导入整个库(包含Core，Gson和Jackson)，项目会默认选择第一个(依据Gson和Jackson的排列位置)。
