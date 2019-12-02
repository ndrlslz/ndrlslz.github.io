title: Salt UI Demo
date: 2016-01-16 00:13:25
tags:
---

因为之前学习了saltstack，所以有个简单的想法，想写个salt management UI，这样就不用输入繁琐的命令，方便管理和运维。
由于没多少时间，就做了个简单的demo，仅仅写了登陆，key管理，远程执行minion的cmd命令的功能，在这里记录下。


### 涉及框架
demo采用springMVC + velocity + bootstrap框架。

--------

<!-- more -->
### 方法介绍
原理主要还是通过调用salt-api来实现各种操作。

#### 1.Key management
salt中， **Wheel** module是对key操作的一个封装， 通过调用wheel模块的api，实现key的accept，delete，reject，list等操作。

Wheel api参考： [Wheel API](https://docs.saltstack.com/en/latest/ref/wheel/all/salt.wheel.key.html#module-salt.wheel.key)

调用方式，这里用list key举例。

```
curl -X POST -Ssk -i https://<master ip>:8000/login \
-H "Accept: application/x-yaml" -d username=<your username> \
-d password=<your password> -d eauth=pam | awk '/token/ {printf $2}' > ~/token

curl -Ssk -i https://<master ip>:8000/ \
-H "X-Auth-Token: `cat ~/token`" -H "Accept: application/x-yaml" \
-d client='wheel' -d fun='key.list_all'
```

#### 2.Run cmd
**Execution** module是远程执行一些function的模块，其中，cmd.run这个function就是执行minion的shell命令了。当然，其他function也可以执行，我的demo里只实现了cmd.run这个function。

Execution module参考： [Execution module](https://docs.saltstack.com/en/latest/ref/modules/all/index.html)
cmd.run function参考: [cmd.run funciton](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.cmdmod.html#module-salt.modules.cmdmod)

调用方式举例：

```
curl -X POST -Ssk -i https://<master ip>:8000/login \
-H "Accept: application/x-yaml" -d username=<your username> \
-d password=<your password> -d eauth=pam | awk '/token/ {printf $2}' > ~/token

curl -Ssk -i https://<master ip>:8000/ \
-H "X-Auth-Token: `cat ~/token`" -H "Accept: application/x-yaml" \
-d client='local' -d fun='cmd.run' -d tgt='minion_id' -d arg='df -h'
```

![cmd运行结果](/uploads/cmd.jpg)

-------

### UI
然后就是调用api来实现功能，展示页面了。
调api部分，我比较懒，直接用了我们项目的httpclient库了。
页面部分，我从bootstrap官网下载的页面来用。

**Key Management UI**

![key management ui](/uploads/key_ui.jpg)

**Cmd Run UI**

![cmd run ui](/uploads/cmd_ui.jpg)

-------

### demo配置
如果要运行demo，需要：
1.salt-api正确配置，参考： [salt-api配置](http://www.saltstack.cn/projects/cssug-kb/wiki/salt-api-deploy-and-use)
2.salt user权限正确配置，可参考上一篇文章。
3.demo中，classpath下account.properties，需要配置ip_url,username和password。


代码地址： [demo代码](https://github.com/ndrlslz/SaltManagement)