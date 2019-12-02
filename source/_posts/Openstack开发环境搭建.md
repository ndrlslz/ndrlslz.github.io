title: Openstack开发环境搭建
date: 2016-05-15 00:53:51
tags:
---


## 环境准备
我使用的虚机是VirtualBox安装CentOS 7
安装2张网卡。一个NAT网卡， 用于和外网通信，一个Host only网卡，用于和宿主机通信。

<!-- more -->

---

## 安装devstack
1. 下载devstack源码
```
git clone https://github.com/openstack-dev/devstack.git
```
2. 创建stack用户
```
devstack/tools/create-stack-user.sh
```
3. 将pypi源修改成豆瓣源
```
mkdir ~/.pip
cat > ~/.pip/pip.conf <<EOF
[global]
index-url = https://pypi.douban.com/simple/
EOF
```
4. 配置local.conf文件
详细配置信息参考[local.conf配置](http://docs.openstack.org/developer/devstack/configuration.html)， 我这里就尽量用最小化的配置了，除了把nova-network禁用了，启用了neutron。
```
# vim local.conf

[[local|localrc]]

HOST_IP=10.20.0.210
GIT_BASE=https://github.com

ADMIN_PASSWORD=openstack
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
SERVICE_TOKEN=$ADMIN_PASSWORD

DISABLED_SERVICES=n-net
ENABLED_SERVICES+=,q-svc,q-agt,q-dhcp,q-l3,q-meta,q-metering,neutron
```
5. 开始安装
```
su - stack
devstack/stack.sh
```
6. 出现下面提示就算安装完成
```
This is your host IP address: 10.20.0.210
This is your host IPv6 address: ::1
Horizon is now available at http://10.20.0.210/dashboard
Keystone is serving at http://10.20.0.210:5000/
The default users are: admin and demo
The password: openstack
```
7. 装完之后记得打个snapshot。

---

## Openstack进程、日志信息
在调试之前，先了解下openstack的组件的进程信息和日志信息。

### screen
通过devstack安装的openstack，是通过screen来保持各个组件的会话的。
`screen -ls`可以查看这个会话，通过`screen -x stack`进入这个会话。
进去之后可以看到各个组件正在运行的进程信息。 最下面是组件列表。
通过`ctrl+a+n`显示下一个会话，`ctrl+a+p`显示上一个会话。`ctrl+a+d`退出会话。
通过`ctrl+c`中断当前组件的进程，再按`↑`来重新启动当前组件。

### 日志
如果local.conf没有配置日志路径的话，默认日志信息在/opt/stack/logs里面。

---

## 本地调试
调试openstack的代码可以通过本地调试和远程调试，选择其中适合自己的一种就好，先说说本地调试的方法。

大致思路是将IDE安装到虚机中(可选)，通过python的pdb来进行调试。

### 安装Pycharm(可选)
1. 下载linux版的pycharm到虚机中，进行安装。
2. 我使用的是Xshell去连接虚机，在Xshell的配置中启用Xmanager。
![xmanager图](/uploads/xmanager.jpg)
3. 下载Xmanager进行安装。
4. 在虚机中安装xauth
```
# yum install -y xauth
```
5. 重新XShell，使Xmanager生效，启动pycharm
```
# charm
```
### pdb调试
使用pycharm的将openstack源代码(`/opt/stack`)导入， 没用pycharm的直接用vim去编辑源代码就好。

pdb调试命令:

| 命令        | 解释                       |
| ---------   | ------------               |
| list或l     | 查看附近行的代码           |
| next或者n   | 执行下一行                 |
| step或者s   | 进入函数                   |
| return或者r | 执行代码直到从当前函数返回 |
| pp          | 打印变量值                 |
| quit或q     | 中止并退出                 |


在需要打断点的地方，加代码：
`import pdb`
`pdb.set_trace()`

尝试修改源码进行调试
以neutron的openvswitch agent代码为例。
`/opt/stack/neutron/neutron/plugins/ml2/drivers/openvswitch/agent/main.py`
```
def main():
    print("-----------------")
    import pdb
    pdb.set_trace() # 打断点
    print("-----------------")
    
    common_config.init(sys.argv[1:])
    driver_name = cfg.CONF.OVS.of_interface
    mod_name = _main_modules[driver_name]
    mod = importutils.import_module(mod_name)
    mod.init_config()
    common_config.setup_logging()
    n_utils.log_opt_values(LOG)
    mod.main()
```
然后在screen中找到q-agt会话，重启这个neutron-openvswitch-agent进程(`ctrl+c中断进程, ↑获得启动命令`)。 这样就进入到了打断点的地方。
![pdb_1图](/uploads/openstack_pdb_1.jpg)

然后就用pdb命令进行调试。
![pdb_2图](/uploads/openstack_pdb_2.jpg)

---

## 远程调试
大致思路是利用python的remote debug功能，通过宿主机的IDE来调试虚机里的openstack代码。

1. 虚机和宿主机代码同步
我是用的windows版的sshfs软件，将虚机上的openstack源码映射到宿主机上来。
2. 用宿主机的pycharm打开openstack的源码。
3. 配置python remote debug。
![openstack_remote_debug图](/uploads/openstack_remote_debug.jpg)
4. 启动remote debug，打断点
![openstack_remote_debug2图](/uploads/openstack_remote_debug2.jpg)
5. 修改eventlet启动参数。
`/opt/stack/neutron/neutron/common/eventlet_utils.py`
```
def monkey_patch():
    if os.name == 'nt':
        # eventlet monkey patching the os and thread modules causes
        # subprocess.Popen to fail on Windows when using pipes due
        # to missing non-blocking IO support.
        #
        # bug report on eventlet:
        # https://bitbucket.org/eventlet/eventlet/issue/132/
        #       eventletmonkey_patch-breaks
        eventlet.monkey_patch(os=False, thread=False)
    else:
        eventlet.monkey_patch(os=False, socket=True, time=True,
                              thread=False)
```
6. 在screen中重启进程，发现pycharm进入了调试模式。 就可以进行调试了。

---

## 单元测试
利用tox来运行项目中的单元测试，每个组件已经默认配置了tox.ini。

1. 安装依赖包
```
sudo yum install python-devel openssl-devel python-pip git gcc libxslt-devel mysql-devel postgresql-devel libffi-devel libvirt-devel graphviz sqlite-devel
```
2. 在某个组件中运行tox，例如在neutron中(`/opt/stack/neutron`)
> tox -e py27      #对neutron项目运行所有单元测试
> tox -e py27,pep8 #对neutron运行单元测试和pep8测试
> tox -e py27 neutron.tests.unit.api  #对neutron中的某个目录运行单元测试
> tox -e py27 neutron.tests.unit.api.test_api_common #对某个文件运行单元测试

---

## 集成测试
主要是黑盒测试，通过请求Rest API的方式进行测试。
利用tempest项目进行测试。
```
# 运行某个目录的集成测试
# /opt/stack/tempest/run_tempest.sh -N tempest.api.compute.images
```

---

## 参考文档
[openstack开发环境](http://bingotree.cn/?p=687)
[openstack测试](https://github.com/yongluo2013/osf-openstack-training/blob/master/installation/How-to-setup-openstack-development-environment.md)









