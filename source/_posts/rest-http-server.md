title: rest http server
date: 2018-09-26 20:30:47
tags:
---

restHttpServer是一个基于Netty的轻量级的HTTP Server，旨在RESTful API的开发。

## Features
* 基于NIO模式
* 根据PATH和Method的路由
* 支持PATH的正则表达式匹配
* 支持PATH参数
* 支持HTTP1.1基本的Keepalive
* 默认异常处理，返回可读的Json response

<!-- more -->

## 快速开始
运行下面的代码来启动一个HTTP Server。
```
RouterTable routerTable = new RouterTable();

RestHttpServer
        .create()
        .requestHandler(routerTable)
        .listen(8080);
```
你可以定义一些路由。
```
routerTable.get("/hi").handler(context -> context.response().setBody("hello world"));
```

## 详细文档
`RouterTable`是restHttpServer的一个核心概念，它存储了所有的路由。
路由表示`path & method`和`handler`的映射。

### PATH & METHOD
RouterTable支持基本的HTTP METHOD，它可以通过不同的path和method来分发请求，下面是一些例子。
```
//Support different http method
routerTable
        .get("/hi").handler(context -> { })
        .post("/hi").handler(context -> { })
        .delete("/hi").handler(context -> { })
        .put("/hi").handler(context -> { })
        .patch("/hi").handler(context -> { });

//Support path parameters
routerTable.get("/customers/{id}").handler(context -> { });

//Regular expressions can be used to match path.
routerTable.get("/hey.*").handler(context -> { });
```

### Handler
传入Handler里面的参数是RouterContext，其中存储了Request和Response的信息，下面是一些使用的例子。
```
routerTable.get("/customers/{id}/contacts").handler(context -> {
    HttpServerRequest request = context.request();
    HttpServerResponse response = context.response();

    request.getQueryParams();              //retrieve query parameters
    request.getPathParams().get("id");     //retrieve path parameter
    request.headers().get("Content-Type"); //retrieve headers
    request.getBodyAsString();             //retrieve request body

    response.setBody("hello, world}");     //set response body
    response.headers().set("key", "value");//set header
});
```

## CRUD Example
增删改查的例子：[here](https://github.com/ndrlslz/restHttpServer/tree/master/examples)

## Benchmark

Mac(Intel Core i7/2.2GHz)
```
wrk -t4 -c150 -d30s http://localhost:10080/test

Running 30s test @ http://localhost:10080/test
  4 threads and 150 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.39ms  423.62us  36.46ms   95.61%
    Req/Sec    15.52k     1.22k   51.71k    95.50%
  1855064 requests in 30.10s, 187.53MB read
Requests/sec:  61627.08
Transfer/sec:      6.23MB
```

## 代码地址
[Github](https://github.com/ndrlslz/restHttpServer)
