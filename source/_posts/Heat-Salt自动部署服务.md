title: Heat+Salt自动部署服务
date: 2016-01-09 14:45:06
tags:
---


记录下在PaaS层自动部署服务的学习过程。

### 整体流程
Heat来做自动编排，Salt来安装服务和对虚机的管理。
>* 1.Create vm
>* 2.Install minion
>* 3.Master accept minions's key
>* 4.Install service

--------

<!-- more -->
### 分析和编码

#### Heat VS Cloudify
因为之前项目也用过cloudify，所以大致的比较下。Heat和cloudify都是PaaS层的，但它们的规模不同，可以说cloudify要强大的多。
**Heat**更趋近于Project，它是openstack众多组件中的一个，目的是为了对资源(计算，网络，存储等等)的自动化操作，所以如果要有一套完整的体系，需要自己结合其他的组件，比如本文中与Salt结合。
**Cloudify**更趋近于一套完整的解决方案，它自带了很多的plugin，可以集成很多功能，比如：openstack，Salt，Chef，Docker等等。它还带有Monitor和Scala的功能。

下面开始部署服务

#### 1. Create vm
首先用heat template创建虚机。

```
minion_server:
  type: OS::Nova::Server
  properties:
    networks:
      - port: { get_resource: minion_server_port }
    name: { get_param: name }
    image: { get_param: image }
    flavor: { get_param: flavor }
    config_drive: 'true'
    user_data_format: SOFTWARE_CONFIG
    user_data: { get_resource: minion_server_mime }
```

#### 2. Install minion
然后通过user_data注入脚本来安装salt-minion。

```
install_minion:
  type: OS::Heat::SoftwareConfig
  properties:
    config: | 
      #!/bin/bash
      rpm -Uvh <your salt-minion repo address>
      chkconfig salt-minion on
      echo "master: <your master ip>" > /etc/salt/minion
      echo "id: `hostname -s`" >> /etc/salt/minion
      salt-minion -d 
```

#### 3. Accept key
这里需要用到salt-api和wheel模块。
**salt-api**提供了对minions，jobs，events等等的Rest操作，以及远程执行模块的功能，这里我们需要远程执行wheel模块。
**wheel**模块是对key的一个封装。用于执行对key的list，accept，reject，delete等操作。

其中，在配置salt-api的时候，需要注意配置请求api的用户的权限。
```
external_auth:
  pam:
    user:
      - .*
      - '@wheel'
      - '@runner'
```

在调api的时候，机制和keystone差不多，需要先调/login获得token，再用token去调相应的api,这里是执行wheel模块来accept key。

```
accept_key:
  type: OS::Heat::SoftwareConfig
  properties:
    config: |
      curl -Ssk https://<master ip>:8000/login -H 'Accept: application/x-yaml' \
      -d username=<your username> -d password=<your password> \ 
      -d eauth=pam | awk '/token/ {printf $2}' > ~/token
      curl -Ssk https://<master ip>:8000/ -H 'Accept: application/x-yaml' \
      -H "X-Auth-Token: `cat ~/token`" -d client='wheel' \
      -d match=`hostname -s` -d fun='key.accept'
```


#### 4. Install service
最后就是安装服务了，这里需要用到salt的Reactor system。
**Reactor**是基于Event system的，在执行操作的时候，会通过zeroMQ PUB产生event，而reactor就是监听这些event，然后执行相应的操作。

这个命令在master上运行可以看到产生的实时的event：
```
salt-run state.event pretty=True
```


api中，/hook用于产生event，比如调用/hook/states/state ,就会产生tag为/salt/netapi/hook/states/state的event，我们要做的就是监听这个event,去执行state.sls文件。
```
reactor:
  - '/salt/netapi/hook/states/state':
    - /srv/salt/reactor/states/state.sls
```

这个state.sls里写的是接收参数，并用local.state.sls来执行，参数里面指定tgt(即minion_id)和args(即要执行的state文件)。
```
{% set postdata = data.get('post', {}) %}

run_state:
  local.state.sls:
    - tgt: {{postdata.tgt}}
    {% if "matcher" in postdata %}
    - expr_form: {{postdata.matcher}}
    {% endif %}
    {% if "args" in postdata %}
    - arg:
      - {{postdata.args}}
    {% endif %}
```

然后在heat template中调api来产生这个event。

```
install_service:
  type: OS::Heat::SoftwareConfig
  properties:
    config: |
      curl -Ssk https://<master ip>:8000/hook/states/state \
      -H 'Accept: application/x-yaml' -H "X-Auth-Token: `cat ~/token`" \
      -d tgt=`hostname -s` -d args='memcached_v14'
```

这里的memcached_v14是在master上写好的state文件，用于安装memcache。这样，就可以通过运行heat template来自动安装相应的服务了。

Heat加Salt进行服务自动部署的流程大致就是这样。











































