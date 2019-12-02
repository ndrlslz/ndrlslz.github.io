title: Openstack neutron学习
date: 2016-01-20 22:24:15
tags:
---

最近在学习openstack neutron的东西，记录下自己的一些理解。

### 网络基础知识

**Switches & Vlan**
交换机的作用是来连接设备，实现互通的。network host之间通过交换机连接起来，当host A第一次向host B发送帧数据时，先广播出去，而MAC地址相符的host B便接受到数据，并返回给A，完成通信。这时交换机也会自学习MAC地址和Port的对应关系，下次便可直接向对应MAC地址的Port发送帧数据。

那交换机中进行隔离是通过划分vlan的方式，openstack中也是通过这种方式来隔离tenant的network。network属性的segmentation_id即为vlan_id,如果network不share的话，就只有这个tenant可以访问。

当交换机的某个vlan的port被占完后，比如交换机A的vlan10的port被占完了，需要用到第二个交换机B，也给B划分vlan10，然后将A和B连接起来，连接的端口设置为trunk port，这样从trunk port出去的数据会加上vlan tag的标志，只有vlan tag一样的才能收到。

<!-- more -->
**IP**
二层网络通过MAC来寻址，三层通过IP来寻址。
IP由2部分组成，network number和host idertifier。对一个vlan而言，如果两个ip的network一样的话，就表示它们在同一个subnet子网中，它们就可以直接通信。
假设IP为192.168.1.2，前24位为network number，则netmask可表示为：
1) 255.255.255.0
2) 192.168.1.2/24
而subnet的CIDR表示为：192.168.1.0/24

对于不同network的通信，需要通过网关或路由。假设host A向host B发送数据包，host A查询自己的route table，把数据包发送到相应的网关(这是表示路由器)，然后路由器再查询自己的路由表，再把数据发到对应的host B去。

**DHCP**
host从network动态获取ip通过dhcp协议，openstack是使用dnsmasq工具，通过neutron dhcp agent来分配ip，从日志 /var/log/daemon.log可以查到分配流程如下：
> 1.The client sends a discover (“I’m a client at MAC address 08:00:27:b9:88:74, I need an IP address”)
> 2.The server sends an offer (“OK 08:00:27:b9:88:74, I’m offering IP address 10.10.0.112”)
> 3.The client sends a request (“Server 10.10.0.131, I would like to have IP 10.10.0.112”)
> 4.The server sends an acknowledgement (“OK 08:00:27:b9:88:74, IP 10.10.0.112 is yours”)

**ARP**
即IP地址和MAC地址转换协议。假设host A向host B发送数据包，但不知道B的MAC地址，则A在network中广播一个ARP请求，流程如下：

> host A To: everybody (ff:ff:ff:ff:ff:ff). I am looking for the computer who has IP address 192.168.1.7. Signed: MAC address fc:99:47:49:d4:a0.

> host B To: fc:99:47:49:d4:a0. I have IP address 192.168.1.7. Signed: MAC address 54:78:1a:86:00:a5.

然后A就可以和B在二层通信了，同时A会记录下B的IP和MAC的映射关系。
可以通过arp -n查看。

-----------

### openstack网络知识

**tunnel technology**
openstack的网络实现方式有flat，vlan，gre，vxlan
*flat* :即所有的设备都连接到同一交换机上，可以互相通信。
*vlan* :由于flat容易产生广播风暴，所以引入vlan，在二层进行vlan划分，隔离网络。
*gre & vxlan* ：由于vlan的个数有限，只能有4094个，对公有云来说不够。所以引入gre，vxlan。
gre和vxlan是三层的隧道技术。通过在三层重新封装数据包，在节点之间创建隧道，通过UDP进行传输。

**open vswitch**
open vswitch是开源的虚拟交换机的一种实现。
ovs-vsctl show查看网桥

```
#ovs-vsctl show
3916f0fb-cc92-428f-bde4-5f83cc13205e
    Bridge br-ex
            Port "br-ex--br-eth1"
                trunks: [0]
                Interface "br-ex--br-eth1"
                    type: patch
                    options: {peer="br-eth1--br-ex"}
            Port br-ex
                Interface br-ex
                    type: internal
    Bridge br-int
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
```
上图中，br-ex为和外部物理网络相连的网桥，br-int为集成网桥，宏观上来看，虚机，路由，dhcp等等的Port都连在这个br-int网桥上。


**Network namespace**
Network namespace是linux内核支持的，对网络的隔离机制，只有同一namespace的网络才可以互相看到。
这是openstack neutron实现机制的重要方式。
1. 创建各种网络资源，比如dhcp，router，lbaas等，实际上是创建它们各自的namespace。而它们各自的namespace中有自己的网络接口。
2. 然后通过open vswitch创建的网桥，将资源的namespace中的网络接口与网桥进行连接，实现通信。

以路由器为例，现有外网subnet A: 172.16.0.0/24, 内网subnet B:192.168.111.0/24,如果要让两个网络通信，需要创建个router来连接两个网络。
现在网络拓扑图如下：
![网络拓扑](/uploads/router_namespace.jpg)

当网络创建好后，查看现有的namespace。
```
#ip netns list
qrouter-f4b08cfc-52fd-4515-bc75-73ae5bb5b440
```

创建router实际上是创建了qrouter+networkId的一个namespace，然后再看这个namespace的网络接口。
```
#ip netns exec qrouter-f4b08cfc-52fd-4515-bc75-73ae5bb5b440 ip a
16: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
17: qg-f16db07c-77: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether fa:16:3e:26:3f:c4 brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.130/24 brd 172.16.0.255 scope global qg-f16db07c-77
    inet6 fe80::f816:3eff:fe26:3fc4/64 scope link 
       valid_lft forever preferred_lft forever
18: qr-675b6149-87: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether fa:16:3e:b2:89:91 brd ff:ff:ff:ff:ff:ff
    inet 192.168.111.1/24 brd 192.168.111.255 scope global qr-675b6149-87
    inet6 fe80::f816:3eff:feb2:8991/64 scope link 
       valid_lft forever preferred_lft forever
```

这个router的namespace中有qg-f16db07c-77和qr-675b6149-87两个接口。然后再看下ovs网桥的信息。

```
#ovs-vsctl show
3916f0fb-cc92-428f-bde4-5f83cc13205e
    Bridge br-ex
        Port "qg-f16db07c-77"
            Interface "qg-f16db07c-77"
                type: internal
        Port "br-ex--br-eth1"
            trunks: [0]
            Interface "br-ex--br-eth1"
                type: patch
                options: {peer="br-eth1--br-ex"}
        Port br-ex
            Interface br-ex
                type: internal
    Bridge br-int
        fail_mode: secure
        Port "qr-675b6149-87"
            tag: 1
            Interface "qr-675b6149-87"
                type: internal
        Port "tap10a26320-68"
            tag: 1
            Interface "tap10a26320-68"
                type: internal
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
```

可以看到，qg-f16db07c-77接口接在br-ex网桥上，qr-675b6149-87接口接在br-int网桥上，所以通过这个router，把集成网桥和外网网桥连了起来。那么这个subnet就可以和外网通信了，和拓扑图的效果一样。

其中，这个subnet和外网通信的话，还涉及到NAT，查看这个router的iptables，其中有一条SNAT。
```
#ip netns exec qrouter-f4b08cfc-52fd-4515-bc75-73ae5bb5b440 iptables -t nat -S
-A neutron-l3-agent-snat -s 192.168.111.0/24 -j SNAT --to-source 172.16.0.130
```
所以通过这个router，source是192.168.111.0/24这个subnet的话，会改成router的gateway的ip。并且router会记录修改的信息，这样从外网回来的数据包就能正确找到地址，先到router，再由router到source。

-----------

### openstack neutron框架

下图是compute节点的网络框架图，网络类型是vlan类型。
![compute network](/uploads/compute_node_network_big.png)



我们换成下面这个简单点，容易理解的图来思考下过程。
![compute network two](/uploads/compute_node_network_small.png)

1、 首先我们在openstack中创建个虚机，然后找到虚拟机的定义文件，查看bridge部分。
```
#cat /var/lib/nova/instances/71ee2b62-8585-4a22-943d-55497bf37df9/libvirt.xml
<interface type="bridge">
  <mac address="fa:16:3e:04:37:b0"/>
  <model type="virtio"/>
  <driver name="qemu"/>
  <source bridge="qbre90692e9-74"/>
  <target dev="tape90692e9-74"/>
</interface>
```
可以看到虚拟通过 'tape90692e9-74' 接口连接到了 'qbre90692e9-74' 这个网桥上。
对应上图，qbre90692e9-74这个网桥对应图中的Linux Bridge，而Port tap也和虚机的网卡连接起来。这样就走通了instance和linux bridge的这条路。


2、 然后在compute节点上查看linux bridge。
```
#brctl show
bridge name     bridge id           STP enabled     interfaces
qbre90692e9-74  8000.8a78413589c7   no              qvbe90692e9-74
                                                    tape90692e9-74
```
发现之前的这个网桥有2个接口，除了和虚机相连的tap接口外，还有个qvb接口。

3、 然后在compute节点上查看open vswitch网桥。
```
#ovs-vsctl show
Bridge br-int
    fail_mode: secure
    Port "qvoe90692e9-74"
        tag: 7
        Interface "qvoe90692e9-74"
    Port br-int
        Interface br-int
            type: internal
    Port patch-tun
        Interface patch-tun
            type: patch
            options: {peer=patch-int}
```
br-int网桥中有个qvo接口，可以看到这个qvo和linux bridge中的qvb接口的后缀是一样的，因为它们是一对veth，它们是连通的。这样的话，linux bridge和br-int也相连了。
所以上图中的Port qvb和Port qvo这条路也走通了。

4、 然后如果网络是vlan的话，ovs网桥中会看到如上图的br-ex和br-int网桥通过一对veth相连，然后br-ex再连接到物理的交换机上。
因为我的实验网络是gre的，我的网桥的br-int是和br-tun相连的，这里就不贴图了。


其中，虚机和ovs的br-int网桥之前隔了个linux bridge，虚机没有直接连上br-int，这样做是因为需要iptables的功能。如果open vswitch直接通过Tap设备和虚机连接的话，Tap设备是无法承载iptables的。所以需要在中间加上一层linux bridge，通过linux bridge来实现iptables。



感觉neutron真的是很复杂，有些东西很难理解。我也只能记录下学习neutron的一点皮毛。

---------

### 参考地址
[openstack文档](http://docs.openstack.org/liberty/networking-guide/intro_basic_networking.html)
[openstack_understand_neutron](https://yeasy.gitbooks.io/openstack_understand_neutron/content/index.html)





