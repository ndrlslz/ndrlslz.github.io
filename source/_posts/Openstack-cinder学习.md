title: Openstack cinder学习
date: 2016-03-28 00:08:50
tags:
---


### 存储基本知识

#### 存储方案

**DAS (Direct-Attached Storage)**
服务器和存储设备直接连接的方式。 每个服务器有自己单独的存储设备，存储之间不共享。类似于机器的硬盘这种方式。这种方式的话，扩展性，利用率，容灾等等都很差，这是早期的实现方式。

**SAN (Storage Area Network)**
服务器和存储设备通过光纤网络连接的方式，服务器共享一个磁盘阵列。当服务器需要存储时，存储设备分配出一块区域，然后服务器通过网络连接上这块存储，再对其进行格式化，成为可以用的文件系统。

**DAS (Network Attached Storage)**
这种方式，存储除了有磁盘阵列外，它有自己的文件系统，相当于自己就是一个提供文件系统的存储服务器，当外部服务器需要访问存储的时候，可以直接通过网络连上存储服务器提供的文件系统来使用。

<!-- more -->
#### LVM
**LVM概念**
LVM 逻辑卷管理，是在物理磁盘和文件系统间抽象出来的一层，用于将物理磁盘抽象成一个大的卷组，然后再在这个卷组上划分逻辑卷。相比于传统的磁盘使用，这样提高了磁盘的利用率和扩展的灵活性。

**创建逻辑卷**
1.首先给虚拟机加一块存储设备/dev/vdb，通过fdisk将其分好区。在选择Hex Code的时候，选择8e,表示是个LVM格式的分区。
```sh
#fdisk /dev/vdb
Disk /dev/vdb: 21.5 GB, 21474836480 bytes
4 heads, 32 sectors/track, 327680 cylinders
Units = cylinders of 128 * 512 = 65536 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00029ba0

   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1   *          17      327680    20970496   8e  Linux LVM
```

2.将这个分区设置成LVM的PV(physical volume),表示它是由LVM管理的物理卷，这个物理卷相比于物理存储设备来说，加了些LVM的参数。
```sh
pvcreate /dev/vdb1
```
3.创建一个卷组，把那个pv加入到这个卷组中，即概念中说的，把多个物理卷抽象成一个大的卷组。
```sh
vgcreate vgGroup /dev/vdb1
```
4.然后就可以从这个卷组中划分逻辑卷了
```sh
lvcreate -L 10G -n lv1 vgGroup
```
5.然后就格式化这个逻辑卷，来进行挂载使用
```sh
#mkfs -t ext4 /dev/vgGroup/lv1
#mount /dev/vgGroup/lv1 /test_fs
#df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/vda1              20G  2.1G   17G  12% /
tmpfs                 499M   12K  499M   1% /dev/shm
/dev/mapper/vgGroup-lv1
                      9.8G   23M  9.2G   1% /test_fs
```

**扩展逻辑卷**
如果vg容量不够，需要新加pv来扩容vg，我们这里容量够，就直接扩展了。
```sh
#lvextend -L 15G /dev/vgGroup/lv1
#resize2fs /dev/vgGroup/lv1
#df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/vda1              20G  2.1G   17G  12% /
tmpfs                 499M   12K  499M   1% /dev/shm
/dev/mapper/vgGroup-lv1
                       15G   25M   14G   1% /test_fs
```

#### iSCSI
iSCSI是一种通过TCP/IP协议来传输SCSI命令和块数据的协议。
它分为Target(服务端)和Initator(客户端)。
大致过程是，Target端暴露出存储设备，Initator端发现这个设备，然后登陆存储设备，来使用。


**Target**
> 1. 准备存储介质(文件，物理磁盘，lvm盘)
> 2. 进行配置，将存储介质暴露出来

**Initator**
> 1. 服务发现Target上能用的存储
> 2. Initator端登陆上Target暴露出的设备

----------

### Openstack cinder

#### 部署cinder
**controller节点**
1. 安装必要的服务，mysql，mq，ntp等等
2. 配置keystone cinder相关部分
3. 安装cinder，启动cinder-api，cinder-scheduler服务

参考： [controller节点配置](http://docs.openstack.org/liberty/install-guide-rdo/cinder-controller-install.html)

**cinder节点**
1. 准备好存储介质，通过lvm将其创建成pv，加入cinder-volumes这个vg。
2. 安装cinder，做好配置，启动cinder-volume服务。

参考： [cinder节点配置](http://docs.openstack.org/liberty/install-guide-rdo/cinder-storage-install.html)

下面这样就算安装成功.
```sh
#cinder service-list
+------------------+------------------------+------+---------+-------+----------------------------+-----------------+
|      Binary      |          Host          | Zone |  Status | State |         Updated_at         | Disabled Reason |
+------------------+------------------------+------+---------+-------+----------------------------+-----------------+
| cinder-scheduler | controller.localdomain | nova | enabled |   up  | 2016-03-21T23:01:15.000000 |        -        |
|  cinder-volume   |        cinder0         | nova | enabled |   up  | 2016-03-21T23:01:19.000000 |        -        |
+------------------+------------------------+------+---------+-------+----------------------------+-----------------+
```

然后就可以管理volume了，但是需要attach的话，还需要装iscsi服务，在cinder节点装iscsi-target，在compute节点装iscsi-Initator。

#### cinder学习

**cinder组件**

[![cinder组件](/uploads/cinder-arch.png)](/uploads/cinder-arch.png)

> cinder-api： WSGI的app， 用来提供Rest ful API， 接受http请求， 经过处理再通过mq或http发送给其他组件。
> cinder-scheduler： 处理从cinder-api过来的请求，进行调度算法，决定在哪个host上进行volume操作
> cinder-volume： 管理块存储设备的服务。
> cinder-backup： 提供备份cinder volume的服务

**scheduler调度**
1. 查看volume host的状态， 只选择state为up的host
2. Filter过滤
    1) AvailabilityZoneFilter， 过滤掉zone与选择的zone不同的host
    2) CapacityFilter， 查看host的剩余空间， 过滤掉剩余空间<所选大小的host
    3) CapabilitiesFilter， 查看host的volume type， 过滤掉与所选的type不同的host
    4) 可配置的其他Filter
3. weight算法
    a) AllocatedCapacityWeigher，优先选择剩余空间最小的
    b) CapacityWeigher，优先选择剩余空间最大的 (默认算法)
    c) GoodnessWeigher，根据host的goodness值(0-100)，优先选择最大的
    d) ChanceWeigher， 随机选择一个host



