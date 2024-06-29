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
## 2.1. 服务端
### 2.1.1. 安装服务端软件
```bash
yum install kmod-zfs libzfs5 libzpool5 zfs
yum install kmod-lustre kmod-lustre-osd-zfs lustre lustre-osd-zfs-mount lustre-iokit lustre-resource-agents
```
也可以使用`rpm --reinstall --replacefiles -iUvh xxxx.rpm`命令离线安装。

### 2.1.2. 加载zfs和lustre内核模块
```
modprobe -v zfs
modprobe -v lustre
```

### 2.1.3. 配置网络
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

### 2.1.4. 部署MGS服务
**创建mgtpool**
```bash
zpool create -f -O canmount=off -o cachefile=none mgtpool /dev/sdb
```
使用`zpool`创建pool池可以同时绑定多个磁盘，并采用raid0模式来存储数据。如果需要对pool扩容，必须使用`zpool add`添加磁盘到指定的pool中。

**创建mgt**
```bash
mkfs.lustre --mgs --backfstype=zfs --reformat mgtpool/mgt
```
`mgtpool/mgt`是`mgtpool`的一个逻辑卷，逻辑卷的数量和容量都是可以通过`zfs`命令控制。

**启动mgs服务**
```bash
mkdir -p /lustre/mgt/mgt
mount -t lustre mgtpool/mgt /lustre/mgt/mgt -v
```
原则上挂载点的名字可以任意取名，建议和mgt名字保持一致。如果忘记mgt的名字。可以通过`zfs list`命令查找。


### 2.1.5. 部署MDS服务
**创建mdtpool**
```bash
zpool create -f -O canmount=off -o cachefile=none mdtpool /dev/sdc
```

**创建mdt**
```bash
mkfs.lustre --mdt --fsname fs00 --index 0 --mgsnode=192.168.3.11@tcp --backfstype=zfs --reformat mdtpool/mdt0
```
`mdtpool/mdt0`是`mdspool`的一个逻辑卷，使用`mount`挂载一个逻辑卷，表示启动一个mds服务。  
如果想要在同一个节点上启动多个mds，则需要在`mdtpool`中再申请一个逻辑卷，此时`--reformat`参数可以省略，`--index`必须递增。  
一个mds可以同时管理多个逻辑卷，只需要在`--reformat`参数后同时指定多个逻辑卷。

**启动mds服务**
```bash
mkdir -p /lustre/mdt/mdt0
mount -t lustre mdtpool/mdt0 /lustre/mdt/mdt0 -v
```

### 2.1.6. 部署OSS服务
**创建ostpool**
```bash
zpool create -f -O canmount=off -o cachefile=none ostpool /dev/sdd
```

**创建ost**
```bash
mkfs.lustre --ost --fsname fs00 --index 0 --mgsnode=192.168.3.11@tcp --backfstype=zfs --reformat ostpool/ost0
```

**启动oss服务**
```bash
mkdir -p /lustre/ost/ost0
mount -t lustre ostpool/ost0 /lustre/ost/ost0 -v
```

&nbsp;
## 2.2. 客户端
### 2.2.1. 安装客户端软件
```bash
yum install kmod-lustre-client lustre-client lustre-iokit
```
也可以使用`rpm --reinstall --replacefiles -iUvh xxxx.rpm`命令离线安装。

### 2.2.2. 加载lustre内核模块
```bash
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

### 2.2.4. 挂载文件系统
```bash
mkdir -p /mnt/fs00
mount -t lustre 192.168.3.11@tcp:/fs00 /mnt/fs00 -v
```

&nbsp;
&nbsp;
# 3. 集群卸载
## 3.1. 服务端
### 3.1.1. 关闭所有的服务
```bash
umount mdtpool/mdt0
umount ostpool/ost0
umount mgtpool/mgt
```
### 3.1.2. 删除所有的逻辑卷
```bash
zfs destroy mgtpool/mgt
zfs destroy mdtpool/mdt0
zfs destroy ostpool/ost0
```
### 3.1.3. 删除所有的pool
```bash
zpool destroy mgtpool
zpool destroy mdtpool
zpool destroy ostpool
```