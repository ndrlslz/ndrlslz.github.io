title: Salt自定义module
date: 2016-03-03 20:06:41
tags:
---

Saltstack自带了很多module供使用，但需要自己实现一些功能的时候，就需要来自定义module了。

### Salt Modules

#### State Modules和Execution Modules
Salt有2种Modules，State Modules和Execution Modules。
1. Execution Modules: 它是通过salt的命令行调用的。当执行相应的execution module的时候，会发送给minion相应的命令，让其马上执行这个命令。
2. State Modules: 它是通过states文件调用的。它更倾向于让minion保持某种状态或结果。和Execution Modules不同的是，当执行State Modules的时候，它会检测minion有没有达到state modules想让它达到的效果，比如pkg.installed，它会检测minion有没有装这个软件，如果没有，才会去安装。

<!-- more -->
#### 查看Modules
salt自带了很多的modules，可以在装有master的机器上查看。

**Execution Modules**:
> 1. salt 'minion-id' sys.list_modules , 这个命令可以查看所有的execution modules。
> 2. salt '<minion-id' sys.list_functions , 这个命令查看所有的execution modules以及modules所拥有的functions。
> 3. salt '<minion-id' sys.doc \(modules or functions) , 这个命令可以查看具体某个execution module或其中某个function的具体用法。

**State Modules**
> 1. salt 'minion-id' sys.list_state_modules , 这个命令可以查看所有的state modules。
> 2. salt 'minion-id' sys.list_state_functions , 这个命令可以查看所有的state modules以及其拥有的functions。
> 3. salt 'minion-id' sys.state_doc , 这个命令可以查看具体某个state module或其中某个function的具体用法。

--------

### Modules写法
在写modules的时候，有几个变量使用较多。
1.\__grains\__ 字典，存储了minion的grains
2.\__pillar\__ 字典，存储了minion的pillar
3.\__salt\__ 字典，存储了minion能用的execution modules

**Execution Modules**
因为execution modules的定位是用来让minion执行命令，所以通常用它来执行shell命令，执行各种操作等等。
举个例子，自定义module的时候，经常调用cmd 这个module。
\__salt__['cmd.run']\(command) , 执行shell命令，返回shell执行的字符串结果。

\__salt__['cmd.run_all']\(command) , 执行shell命令，返回一个字典:
> {
>   'pid': int, #执行shell的process id。
>   'retcode': int, #执行成功，返回0，失败返回非0。
>   'stderr': string, #执行失败返回的字符串。
>   'stdout': string, #执行成功返回的字符串。
> }

参考文档：
[salt官方文档](https://docs.saltstack.com/en/latest/ref/modules/index.html)
[salt自带execution modules源码](https://github.com/saltstack/salt/tree/develop/salt/modules)

**State Modules**
state module通常用来调用自定义的或自带的execution module来让minion达到想要的状态和结果。
和execution module写法不同的是，execution module没有规定返回结果的格式，state module规定了它的返回结果格式。必须是一个包含下列key/value的字典。
> {
>   'name': string, #执行的state module的name。
>   'changes': dict, #执行完module的变化。
>   'result': True | False | None,
>   'comment': string, #执行结果的摘要
> }

参考文档：
[salt官方文档](https://docs.saltstack.com/en/latest/ref/states/writing.html)
[salt自带state modules源码](https://github.com/saltstack/salt/tree/develop/salt/states)

---------

### 练习Modules编写
由于最近需要写IBM Websphere Application Server的salt，需要自定义modules来实现一些功能。所以练习下modules的编写。

WAS的集群安装中，需要实现start和stop集群中的dmgr，nodeagent和server，这里贴下学习写的start部分。

**Execution Modules**
```python
import logging
import salt
from salt.exceptions import SaltInvocationError

log = logging.getLogger(__name__)


def start_process(type, profile_path, server_name=None):
    """
    Start the dmgr, node agent or server.
    :param type: dmgr | node | server
    :param profile_path: profile path of server. i.e: /opt/IBM/WebSphere/AppServer/profiles/dmgr01
    :param server_name: server name. i.e: server01

    CLI Example:
    .. code-block: bash
        salt '*' websphere.start_server dmgr /opt/IBM/WebSphere/AppServer/profiles dmgr01
    """
    log.info("type: {0}".format(type))
    log.info("profile_path: {0}".format(profile_path))
    log.info("server_name: {0}".format(server_name))

    pre_command = profile_path + '/bin/'
    if type == 'node':
        command = pre_command + 'startNode.sh'
    elif type == 'dmgr':
        command = pre_command + 'startManager.sh'
    elif type == 'server':
        if not server_name:
            log.error('server_name is null!')
            raise SaltInvocationError('server_name can not be null when type is server')
        command = pre_command + 'startServer.sh ' + server_name
    else:
        log.error('param type error.')
        raise SaltInvocationError('param type error, please select dmgr, node or server.')

    out = __salt__['cmd.run_all'](command, python_shell=True)
    return out
```


**State Module**
```python
import logging
import salt
from salt.exceptions import SaltInvocationError

log = logging.getLogger(__name__)


def start_server(type, profile_path, server_name=None, **kwargs):
    """
    Start the dmgr, node agent or server.
    :param type: dmgr | node | server
    :param profile_path: profile path of server. i.e: /opt/IBM/WebSphere/AppServer/profiles/dmgr01
    :param server_name: while type is server, need server name. i.e: server01

    CLI Example:
    .. code-block: yaml

    websphere.start:
      - type: dmgr
      - profile_path: /opt/IBM/WebSphere/AppServer/profiles/dmgr01
    """
    ret = {
        "name": "start_server",
        "changes": {},
        "result": False,
        "commit": ''
    }
    check_out = _check_server_running(profile_path)
    if check_out['retcode'] == 0:
        ret['changes'] = {"changes": type + " is always running."}
        ret['result'] = True
        ret['comment'] = check_out['stdout']
        return ret

    start_out = __salt__['websphere.start_process'](type, profile_path, server_name)
    if start_out['retcode'] == 0:
        ret['changes'] = {"changes": type + " started successfully."}
        ret['result'] = True
        ret['comment'] = start_out['stdout']
        return ret

    else:
        ret['changes'] = {"changes": type + " started failed."}
        ret['result'] = False
        ret['comment'] = start_out['stdout']
        return ret


def _check_server_running(profile_path):
    """
    Check server running or not.
    :param type: dmgr | node | server
    :param profile_path: profile path of server. i.e: /opt/IBM/WebSphere/AppServer/profiles/dmgr01
    :param server_name: while type is server, need server name. i.e: server01
    """
    command = 'ps aux | grep -e "{0}" | grep -v grep'.format(profile_path)
    out = __salt__['cmd.run_all'](command, python_shell=True)
    return out
```
























