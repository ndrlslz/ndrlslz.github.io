title: 知乎日报SDK
date: 2016-10-10 21:39:54
tags:
---

## 知乎日报SDK

之前逛知乎，有个问题是"有什么有趣的API接口"，就顺便找了个[知乎日报的API](https://github.com/izzyleung/ZhihuDailyPurify/wiki/%E7%9F%A5%E4%B9%8E%E6%97%A5%E6%8A%A5-API-%E5%88%86%E6%9E%90)来做了个Java版的SDK。

话说用Retrofit2这个HttpClient库真是方便，几乎都不用写啥代码了。

---
<!-- more -->


## 项目地址
[知乎日报SDK](https://github.com/ndrlslz/zhihuDaily-sdk)

---
## 快速了解

**获取ZhihuDailyClient**
```java
//首先，获取ZhihuDaily对象
ZhihuDaily zhihuDaily = ZhihuDailyClient.create();
```

**同步或异步调用**
你可以同步或异步的去调用知乎日报的API。

同步方式，通过调用`execute()`
```
//同步调用
DailyNews dailyNews = zhihuDaily.getLatestNews().execute();
dailyNews.getStories().forEach(System.out::println);
```

异步方式，通过调用`enqueue()`
```
//异步调用
zhihuDaily.getLatestNews().enqueue(new ServiceCallback<DailyNews>() {
    @Override
    public void onResponse(DailyNews object) {
        object.getStories().forEach(System.out::println);
    }

    @Override
    public void onFailure(HttpException exception) {
        System.out.println(exception.getMessage());
    }
});
```

---
## 安装方式
需要使用Java8及以上。

**Maven**
```
<dependency>
    <groupId>com.github.ndrlslz</groupId>
    <artifactId>zhihuDaily-java-client</artifactId>
    <version>0.1.2</version>
</dependency
```

**Gradle**
```
compile 'com.github.ndrlslz:zhihuDaily-java-client:0.1.2'
```

---
## 更多例子
[zhihuDaily-sdk实例](https://github.com/ndrlslz/zhihuDaily-sdk/blob/master/zhihuDaily-java-examples/src/main/java/com/github/ndrlslz/Examples.java)



