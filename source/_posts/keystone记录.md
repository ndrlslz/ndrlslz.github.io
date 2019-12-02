title: keystone记录
date: 2015-12-06 16:03:32
tags:
---

记录下openstack中keystone的安装及基本用法。

### 环境信息

**虚机**
使用virtualBox安装CentOS 7作为节点

**网络**
创建3张网卡
>* management network：10.20.0.0/24
>* private network: 192.168.4.0/24
>* external network: 172.16.0.0/24

***

<!-- more -->
### 安装步骤
1. 创建个虚机作为controller节点，我这里给它2个网卡，一个**management**的网卡，用于controller节点与其他节点的通信，一个**NAT**的网卡，让虚机可以上网。进虚机后先把网络配置好。我设置的controller节点的ip为10.20.0.10.
2. 安装ntp服务，参考 [ntp安装](http://docs.openstack.org/liberty/install-guide-rdo/environment-ntp.html)
3. 安装epel源和openstack的源 参考 [安装源](http://docs.openstack.org/liberty/install-guide-rdo/environment-packages.html)
4. 安装mysql和rabbitmq的服务 参考 [mysql](http://docs.openstack.org/liberty/install-guide-rdo/environment-sql-database.html) [rabbitmq](http://docs.openstack.org/liberty/install-guide-rdo/environment-messaging.html)
5. 然后就可以安装keystone的服务和进行相关配置了，参考 [keystone安装](http://docs.openstack.org/liberty/install-guide-rdo/keystone-install.html)
6. keystone装好后，开始安装keystone service和endpoint 参考 [service，endpoint](http://docs.openstack.org/liberty/install-guide-rdo/keystone-services.html)
7. 最后再创建好tenant,user,role。 参考[create tenant,user,role](http://docs.openstack.org/liberty/install-guide-rdo/keystone-users.html)

之后再创建个脚本文件，方便以后命令行操作。

```
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
```

这是官网的脚本内容，但因为keystone命令行客户端在v3版被弃用了，所以在命令行无法使用keystone v3的认证服务，这样用keystone开头的命令会出错，所以需要把版本改成v2.0.

```
export OS_AUTH_URL=http://controller:35357/v2.0
```
这样就可以使用keystone CLI了。
然后到这些，keystone的基本安装就结束了，一些驱动的配置都是按照官网安装步骤的配置来配的，例如认证方式采用UUID，token的持久化用的memcache。

### keystone认证流程


[![keystone流程图](/uploads/Keystone-workflow.png)](/uploads/Keystone-workflow.png)


#### 认证方式
之前提到的认证方式，现在keystone有UUID，PKI，ZPKI三种方式，不同点可以参考 [keystone认证方式](https://www.mirantis.com/blog/understanding-openstack-authentication-keystone-pki/)

大概总结一下：
**UUID**
UUID方式通过keystone生成32位的字符串token，然后保存在backend中，并返回给申请者，之后拿到这个token去请求其他的openstack组件，而其他组件会再通过keystone验证这个token是否有效。
这个方式可能会有几个问题：
1. 其他组件都会来keystone进行token验证，keystone很容易成为系统的瓶颈，特别对于多region的情况，所以可以考虑对keystone做HA。
2. 如果backend采用sql，默认不会清理token，所以token数量增大，查询效率会降低。
这里keystone-manage token-flush命令可以把过期token删掉，并把token有效期设为1小时。backend也可以采用memcahce。

**PKI**
PKI方式的keystone相当于一个Certificate Authority，当请求token时，它会生成包含catalog，metadata等的json，然后用自己的密钥和证书来签署，生成token。而其它组件会有自己的一份公钥和证书，对token进行解密得到json串，所以不必再去请求keystone进行验证，它自己就可以进行验证了，降低了keystone的负载。
这个方式也可能会存在问题：
1. pki的本身性能比UUID要低。
2. UUID方式只需要拿32位字符串这个token去请求其他组件即可，而PKI需要拿整个json串的token去请求，数据包很大。

**ZPKI**
ZPKI方式就是对PKI方式的token进行zlib压缩。

#### 具体用法

我这里是采用uuid的方式进行认证。去调keystone的REST API即可。 API参考 [keystone API](http://developer.openstack.org/api-ref-identity-v3.html)

先定义好请求的json串 vim auth.json
```sh
{
    "auth": {
        "identity": {
            "methods": [
                "password"      
            ],          
            "password": {
                "user": {       
                    "id": "ADMIN_USER_ID",
                    "password": "ADMIN_USER_PASSWORD"
                }               
            }           
        },      
        "scope": {
            "project": {
                "id": "ADMIN_PROJECT_ID"
            }           
        }       
    }   
}
```

然后我这里用curl调api
```bash
curl -i -X POST -H "Content-Type: application/json" -d "`cat auth.json`" \
http://10.20.0.10:5000/v3/auth/tokens
```

返回的消息的头部如下
```
HTTP/1.1 201 Created
Date: Thu, 10 Dec 2015 20:42:31 GMT
Server: Apache/2.4.6 (CentOS) mod_wsgi/3.4 Python/2.7.5
X-Subject-Token: d576cfe5c3d54b46a8aa2d9a153979b0
Vary: X-Auth-Token
x-openstack-request-id: req-7d3d627a-c77f-49b0-a93b-755d93ff5646
Content-Length: 1065
Content-Type: application/json
```

这样就得到了keystone返回的token，body体中还有catalog等信息我没截出来。
然后取Header中的X-Subject-Token去向catalog中的某个endpoint发送请求即可。其他组件得到这个token后，再向keystone验证token的有效性，执行相应操作。keystone大致流程就是这样。











