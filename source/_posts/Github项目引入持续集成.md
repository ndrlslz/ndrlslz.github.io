title: Github项目引入持续集成
date: 2016-08-15 19:57:37
tags:
---

## 持续集成

持续集成强调开发人员提交了新代码之后，立刻进行构建、（单元）测试。根据测试结果，我们可以确定新代码和原有代码能否正确地集成在一起。

对于传统方式，往往是开发人员先开发完一个功能，然后在delivery之前，由测试人员进行测试，如果没有问题，再打包发布。而持续集成表示针对于每一个commit，系统都会自动地进行上述步骤，如果集成结果亮红灯，就需要立即修复代码。

可以看到，持续集成缩短了构建和测试的周期，自动化的进行构建，运行测试。既减少了测试人员的工作，又可以让开发人员清楚的看到是哪个commit出现了问题，及时的发现和处理问题。这样使得开发人员的代码是可以直接打包发布的。

---
<!-- more -->

## 引入持续集成

### Travis-CI

Travis CI是一个开源的持续集成的工具，用于自动化运行测试，类似工具的还有jenkins, GO。使用Travis CI是因为它与Github集成很方便。

**步骤**
1. 使用Github账号登陆[Travis CI](https://travis-ci.org)
2. 同步Github的项目到Travis CI, 打开对应项目的开关。
3. 在你的项目根目录创建`.travis.yml`。内容的编写参考[.travis.yml编写](https://docs.travis-ci.com/user/getting-started/)
4. 完成上述步骤就算配置完成了，然后就可以push一个commit来触发travis CI了。
5. 在Travis CI官网,点击build状态图标获取链接，放到项目的README中。

### Codecov

Travis CI的作用是对项目测试的正确性进行反馈，而`codecov`是对测试的覆盖率进行反馈。

**步骤**
1. 使用Github账号登陆[codecov](https://codecov.io)
2. 在`.travis.yml`中加入下面的代码，用于在构建完成后将测试覆盖率报告发给codecov。
```
after_success:
  - bash <(curl -s https://codecov.io/bash)
```
3. 在项目根目录创建`codecov.yml`,内容编写参考[codecov.yml编写](https://github.com/codecov/support/wiki/Codecov-Yaml)
下面是个简单的例子。第一个80%表示每次commit的测试覆盖率不能低于80%，第二个80%表示项目的整体测试覆盖率不能低于80%。
```
coverage:
  precision: 2
  round: down
  status:
    patch:
      default:
        target: 80%
    project:
      default:
        target: 80%

comment:
  layout: "header, diff, uncovered"
  behavior: default
```
4. 在Codecov官网,点击Settings里面的Badge来获取覆盖率图标，放到README中。

最后，引入持续集成的结果如下图:

![集成图](/uploads/ci-ico.jpg)

---
## 感受

当为项目引入持续集成后，我可以明显感觉到一些好处。

* 可以立马知道每一次commit的测试结果，保证代码随时都可以交付，并且对于定位问题很方便，可以清楚的知道是哪次commit的代码出了问题。
* 保证测试代码的覆盖率，因为引入持续集成后，对测试的要求比较高，需要有足够的测试用例。集成测试可以强迫开发者达到相应目标的测试覆盖率。

