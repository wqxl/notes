> 系统版本：Centos 8.4.2150  
> 内核版本：4.18.0-305.3.1.el8.x86_64  
> lustre版本：2.15.3  
> backfstype：zfs  

&nbsp;
# 1. 准备
## 1.1. 防火墙设置
### 1.1.1. 关闭防火墙
```bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```

&nbsp;
## 1.2. selinux设置
### 1.2.1. 关闭selinux
```bash
sed -i.org 's|SELINUX=enforcing|SELINUX=disabled|g' /etc/selinux/config
```

### 1.2.2. 重启机器
```bash
reboot
```

### 1.2.3. 检查selinux状态
```bash
getenforce
```
如果输出结果是disabled，表明selinux已经关闭。  
注：客户端和服务端都需要执行以上所有操作。


&nbsp;
&nbsp;
# 2. 集群部署
## 2.1. 集群规划
```bash
mgt      192.168.3.11     node1
mdt0     192.168.3.11     node1
mdt1     192.168.3.12     node2
ost0     192.168.3.13     node3
ost1     192.168.3.14     node4
```
node1与node2互为主备，node3与node4互为主备。

## 2.2. 服务端
### 2.2.1. 安装服务端软件
```bash
yum install kmod-zfs libzfs5 libzpool5 zfs
yum install kmod-lustre kmod-lustre-osd-zfs lustre lustre-osd-zfs-mount lustre-iokit lustre-resource-agents
```
也可以使用`rpm --reinstall --replacefiles -iUvh xxxx.rpm`命令离线安装。

### 2.2.2. 加载zfs和lustre内核模块
```
modprobe -v zfs
modprobe -v lustre
```

### 2.2.3. 配置网络
lustre集群内部通过LNet网络通信，LNet支持InfiniBand and IP networks。本案例采用TCP模式。

**初始化配置lnet**
```bash
lnetctl lnet configure
```
默认情况下`lnetctl lnet configure`会加载第一个up状态的网卡，所以一般情况下不需要再配置net，可以使用`lnetctl net show`命令列出所有的net配置信息，如果没有符合要求的net信息，需要按照下面步骤添加。

**添加tcp**
```bash
lnetctl net add --net tcp --if enp0s8
```
如果`lnetctl lnet configure`已经将添加了tcp，使用`lnetctl net del`删除tcp，然后用`lnetctl net add`重新添加。

**查看添加的tcp**
```bash
lnetctl net show --net tcp
```

**保存到配置文件**
```bash
lnetctl net show --net tcp >> /etc/lnet.conf
```

**开机自启动lnet服务**
```bash
systemctl enable lnet
```
注：所有的服务端都需要执行以上操作。

&nbsp;
### 2.2.4. 部署MGS服务
#### 2.2.4.1. 节点node1
**创建mgtpool**
```bash
zpool create -f -O canmount=off -o multihost=on -o cachefile=none mgtpool /dev/sdb
```
容灾模式下使用zpool创建pool时，必须要开启了multihost功能支持。multihost要求为每一个host提供不同的hostid，如果不提供，该命令执行失败。在每一个host上执行`zgenhostid $(hostid)`便可以生成不同的hostid。  
使用`zpool`创建pool池可以同时绑定多个磁盘，并采用raid0模式来存储数据。如果需要对pool扩容，必须使用`zpool add`添加磁盘到指定的pool中。

**创建mgt**
```bash
mkfs.lustre --mgs \
--servicenode=192.168.3.11@tcp \
--servicenode=192.168.3.12@tcp \
--backfstype=zfs \
--reformat mgtpool/mgt
```
`mgtpool/mgt`是`mgtpool`的一个逻辑卷，逻辑卷的数量和容量都是可以通过`zfs`命令控制。  
`servicenode`参数指定当前创建的mgt能够在哪些节点上被使用(容灾)。该参数的数量没有限制。可以将多个`servicenode`参数合并成一个，比如上面的参数可以改写成`--servicenode=192.168.3.11@tcp:192.168.3.12@tcp`。

**启动mgs服务**
```bash
mkdir -p /lustre/mgt/mgt
mount -t lustre mgtpool/mgt /lustre/mgt/mgt -v
```

### 2.2.5. 部署MDS服务
#### 2.2.5.1. 节点node1
**创建mdtpool**
```bash
zpool create -f -O canmount=off -o multihost=on -o cachefile=none mdt0pool /dev/sdc
```

**创建mdt**
```bash
mkfs.lustre --mdt \
--fsname fs00 \
--index 0 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--servicenode 192.168.3.12@tcp \
--backfstype=zfs \
--reformat mdt0pool/mdt0
```
如果mgs服务有多个，必须要同时指定多个mgsnode，而且第一个mgsnode必须是primary mgs。  
对于每一个lustre文件系统，mdt index序号必须从0开始，0代表整个文件系统的根目录。  
`mdt0pool/mdt0`是`mdt0pool`的一个逻辑卷，使用`mount`挂载一个逻辑卷，表示启动一个mds服务。如果想要在同一个节点上启动多个mds，则需要在`mdt0pool`中再申请一个逻辑卷，此时`--reformat`参数可以省略，`--index`必须递增。  
一个mds可以同时管理多个逻辑卷，只需要在`--reformat`参数后同时指定多个逻辑卷。

**启动mds服务**
```bash
mkdir -p /lustre/mdt/mdt0
mount -t lustre mdt0pool/mdt0 /lustre/mdt/mdt0 -v
```

#### 2.2.5.2. 节点node2
**创建mdspool**
```bash
zpool create -f -O canmount=off -o multihost=on -o cachefile=none mdt1pool /dev/sdb
```

**创建mdt**
```bash
mkfs.lustre --mdt \
--fsname fs00 \
--index 1 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--backfstype=zfs \
--reformat mdt1pool/mdt1
```
如果mgs服务有多个，必须要同时指定多个mgsnode，而且第一个mgsnode必须是primary mgs。另外对于每一个lustre文件系统，mdt index序号必须从0开始，0代表整个文件系统的根目录。

**启动mds服务**
```bash
mkdir -p /lustre/mdt/mdt1
mount -t lustre mdt1pool/mdt1 /lustre/mdt/mdt1 -v
```

### 2.2.6. 部署OSS服务
#### 2.2.6.1. 节点node3
**创建ostpool**
```bash
zpool create -f -O canmount=off -o multihost=on -o cachefile=none ost0pool /dev/sdd
```

**创建ost**
```bash
mkfs.lustre --ost \
--fsname fs00 \
--index 0 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.13@tcp \
--servicenode 192.168.3.14@tcp \
--backfstype=zfs \
--reformat ost0pool/ost0
```

**启动oss服务**
```bash
mkdir -p /lustre/ost/ost0
mount -t lustre ost0pool/ost0 /lustre/ost/ost0 -v
```

#### 2.2.6.2. 节点node4
**创建ostpool**
```bash
zpool create -f -O canmount=off -o multihost=on -o cachefile=none ost1pool /dev/sdc
```

**创建ost**
```bash
mkfs.lustre --ost \
--fsname fs00 \
--index 1 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.14@tcp \
--servicenode 192.168.3.13@tcp \
--backfstype=zfs \
--reformat ost1pool/ost1
```

**启动oss服务**
```bash
mkdir -p /lustre/ost/ost1
mount -t lustre ost1pool/ost1 /lustre/ost/ost1 -v
```

&nbsp;
## 2.3. 客户端
### 2.3.1. 安装客户端软件
```bash
yum install kmod-lustre-client lustre-client lustre-iokit
```
也可以使用`rpm --reinstall --replacefiles -iUvh xxxx.rpm`命令离线安装。

### 2.3.2. 加载lustre内核模块
```bash
modprobe lustre
```

### 2.3.3. 配置网络
lustre集群内部通过LNet网络通信，LNet支持InfiniBand and IP networks。本案例采用TCP模式。

**初始化配置lnet**
```bash
lnetctl lnet configure
```
默认情况下`lnetctl lnet configure`会加载第一个up状态的网卡，所以一般情况下不需要再配置net，可以使用`lnetctl net show`命令列出所有的net配置信息，如果没有符合要求的net信息，需要按照下面步骤添加。

**添加tcp**
```bash
lnetctl net add --net tcp --if enp0s8
```
如果`lnetctl lnet configure`已经将添加了tcp，使用`lnetctl net del`删除tcp，然后用`lnetctl net add`重新添加。

**查看添加的tcp**
```bash
lnetctl net show --net tcp
```

**保存到配置文件**
```bash
lnetctl net show --net tcp >> /etc/lnet.conf
```

**开机自启动lnet服务**
```bash
systemctl enable lnet
```
注：所有的客户端都需要执行以上操作。

### 2.3.4. 挂载文件系统
```bash
mkdir -p /mnt/fs00
mount -t lustre 192.168.3.11@tcp:192.168.3.12@tcp:/fs00 /mnt/fs00 -v
```