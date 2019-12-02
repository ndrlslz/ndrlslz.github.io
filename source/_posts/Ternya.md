title: Ternya
date: 2016-05-07 22:10:15
tags:
---


## 写tenrya的初衷
现在部门做的项目是个基于openstack的云平台，其中一个模块是需要监听openstack的notification来进行响应，现有后台代码是用Java写的，逻辑是用switch-case判断notification的event_type来调用对应的响应类。可以想象到这段switch-case有多长。

希望用更优雅的方式来做这个事，所以就把这块抽离了出来，写个了简单的python library。
ternya的目的是为了更方便的接收和处理openstack notification。

<!-- more -->

---

## 安装ternya
3种方式都可以：
>pip install ternya
>easy_install ternya
>python setup.py install

---

## 如何填写配置文件
### 配置文件的例子
[example config file](https://github.com/ndrlslz/ternya/blob/master/config.ini)

### 配置信息解释
**project_abspath**
填写你的项目根目录的绝对路径地址。

**packages_scan**
你的项目中，处理notification的方法所在的package。
ternya会扫描你填写的package下的所有py文件，然后注入处理notification的业务逻辑。
意思就是处理notification的逻辑放在哪，就填它所在的package。

举个例子，假如项目目录结构如下：
```
E:\ternya
|-- my_process_one.py
`-- process
   |-- my_process_two.py
   `-- sub_process
       |-- my_process_three.py
`-- process1
   |-- my_process_four.py
```

1. 如果不填，即 packages_scan = 
ternya会扫描项目的所有py文件
2. 如果填process这个package，即 packages_scan = process
tenrya会扫描到my_process_two.py和my_process_three.py 这2个py文件。
3. 如果填process.sub_process这个package，即 packages_scan = process.sub_process
ternya会扫描my_process_three.py 这个py文件。
4. 如果填多个package，之间以分号分割， 即 packages_scan = process.sub_process;process1
tenrya会扫描my_process_three.py和my_process_four.py 这2个py文件

为了内存和速度着想，让tenrya扫描的py文件越少越好，意味着最好只扫描处理notification的那些py文件。

**mq_user**
openstack mq的username

**mq_password**
openstack mq的password

**mq_host**
openstack controller节点的mq的host

**listen_notification**
是否监听这个组件的notification

**mq_consumer_count**
对每个组件而言，启用多少个consumer去监听和处理notification。
比如启动3个consumer：
![multi_consumer](/uploads/consumer_count.jpg)
queue通过轮询的方式将message负载均衡到多个consumer。

---
## 如何运行ternya
1. 通过调用read(path)方法加载你的配置文件。
2. 通过调用work()来启动ternya

如果在项目中用到ternya，需要启用一个进程来启动ternya。
提醒：如果是windows平台，要想启动进程， if __name__ == "__main__"是必须要的。
```java
from ternya import Ternya
from multiprocessing import Process

if __name__ == "__main__":
    ternya = Ternya()
    ternya.read("config.ini")
    process = Process(target=ternya.work)
    process.start()
```

---

## 如何处理notification
如一开始提到的，ternya的主要目的就是更简单的处理notification。
ternya提供了基于注解的方式来处理notification。
**注解使用方法**: @openstack组件名称("event_type")

举个例子，比如要处理开始创建虚机的notification。
```
from ternya import nova

@nova("compute.instance.create.start")
def create_instance_start(body, message):
    """
    Service method of dealing with notification.
    body and message parameter is necessary.

    :param body: notification dict
    :param message: kombu Message class
    """
    print("start to create instance notification.")
    print(body['event_type'])
    print(body)
```
通过注解的方式，ternya会记录event_type和业务方法的映射关系，在这个例子中，ternya会记录下用create_instance_start这个方法来处理'compute.instance.create.start'这个event_type。

ternya的注解也支持**通配符**的方式。
比如要处理删除虚机的notification。
```
from ternya import nova

@nova("compute.instance.delete.*"):
def instance_delete(body, message):
    print("delete instance notification")
    print(body['event_type'])
    print(body)
```
在这个例子中，instance_delete这个方法可以处理2种notification，分别为event_type为"compute.instance.delete.start"和"compute.instance.delete.end"的notification。

ternya目前支持7种openstack组件的注解。
> nova
> cinder
> neutron
> glance
> swift
> keystone
> heat

---

## 如何编写处理notification的方法
ternya中使用了kombu库，采用kombu提供的方式来编写方法，方法需要接收2个参数，body和message。一般只需要用到body即可。
**body**
notification的字典
**message**
kombu.transport.base.Message

ternya默认处理notification的方式是通过log的DEBUG模式输出。

---

## 源码地址
[ternya源码](https://github.com/ndrlslz/ternya)





