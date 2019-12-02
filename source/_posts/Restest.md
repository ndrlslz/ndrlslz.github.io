title: Restest
date: 2018-09-22 20:21:06
tags:
---

Restest是一个针对RESTFul API的自动化测试工具。
它通过编写YAML的方式来实现测试用例，并提供易于理解的输出，可以方便的和CI进行集成。

<!-- more -->

## 基本用法
假设下面是我们需要测试的API。
```
Request: GET http://localhost:8080/customers

Response:
{
  "data": [
    {
      "id": 1,
      "name": "Tom"
    },
    {
      "id": 2,
      "name": "Nick"
    }
  ]
}
```

为了测试这个API，我们可以编写下面的`customers.yml`文件作为测试用例。
```
api:
  name: test customer api
  endpoint: http://localhost
  port: 8080

scenarios:
  - name: retrieve customers
    method: GET
    path: /customers
    headers:
      Accept: application/json
    expect:
      status: 200
      headers:
        Content-Type: application/json
      body:
        data.id: [1, 2]
        data[0].id: 1
        data[0].name: Tom
        data.find { it.name == "Nick" }.id: 2
    variables:
      NickId: data.find { it.name == "Nick" }.id
```

然后来运行它
1. Git clone代码，代码地址：https://github.com/ndrlslz/restest
2. `./gradlew build`
3. `cd build/lib`
4. Run `java -jar restest.jar customers.yml`

运行结果如下图
![restest_result](/uploads/restest_result.png)

## 详细文档
YAML的结构由两部分构成，API和Scenarios。

### API
API部分描述了这个API的基本信息，比如Endpoint，Port。
```
api:
  name: # required - your api's name
  endpoint: # required - http://localhost
  port: # required
  username: # optional - basic auth's username
  password: # optional - basic auth's password
```

#### Scenarios
Scenarios描述了所有的测试用例。
```
scenarios:
  - name: test case one
  ...


  - name: test case two
  ...
```

### Scenarios语法
```
scenarios:
  - name: # required - test case name
    path: # required - /path/to/the/resource
    method: # optional - GET by default
    headers: # optional - set the request headers
      Header-Name: Header-Value
    body: > # optional - set the request body
      {
        "json": "..."
      }
    expect: # optional
      status: # optional - 200 by default
      headers: # optional - expected headers
        Header-Name: Header-Value
      body: # optional - expected body
        id: 1
    variables: # optional - store variables
      Var-Name: Var-Value
```

关键字`name`用来识别不同的test case，请保证唯一性。
`path`, `method`, `headers`, `body`用来构造HTTP请求。
`expect`用来验证HTTP响应, 其中, `expect.body`使用[RestAssured](http://rest-assured.io/)库来验证HTTP响应。
`variables`用于存储变量，可以在之后的Scenarios中通过`${var_name}`使用。

### 命令行参数
`java -jar restest.jar <file location or folder location> [true|false]`

在运行`restest.jar`时，你可以提供2个参数。
1. 路径， 可以是单个yml的路径，也可以是包含多个yml的文件夹的路径。
2. 详细日志(可选参数)，是否开始详细日志，默认为false。

下图是开启详细日志的例子
![restest_detailed_result](/uploads/restest_detailed_result.png)

### 例子
更多例子，可以参考[examples](https://github.com/ndrlslz/restest/blob/master/examples/example.yml)

## 代码地址
[Github](https://github.com/ndrlslz/restest)
