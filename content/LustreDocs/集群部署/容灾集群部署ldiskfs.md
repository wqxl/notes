> 系统版本：Centos 8.4.2150  
> 内核版本：4.18.0-305.3.1.el8.x86_64  
> lustre版本：2.15.3  
> backfstype：ldiskfs  

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
yum reinstall kernel kernel-modules kernel-core kernel-tools kernel-tools-libs
yum install e2fsprogs e2fsprogs-libs
yum install kmod-lustre kmod-lustre-osd-ldiskfs lustre lustre-osd-ldiskfs-mount lustre-iokit lustre-resource-agents
```
也可以使用`rpm --reinstall --replacefiles -iUvh xxxx.rpm`命令离线安装。

### 2.2.2. 加载lustre内核模块
```
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
**创建mgt**
```bash
mkfs.lustre --mgs \
--servicenode=192.168.3.11@tcp \
--servicenode=192.168.3.12@tcp \
--backfstype=ldiskfs \
--reformat /dev/sdb
```
`servicenode`参数指定当前创建的mgt能够在哪些节点上被使用(容灾)。该参数的数量没有限制。可以将多个`servicenode`参数合并成一个，比如上面的参数可以改写成`--servicenode=192.168.3.11@tcp:192.168.3.12@tcp`。

**启动mgs服务**
```bash
mkdir -p /lustre/mgt/mgt
mount -t lustre -U 1a8bdce6-1801-41b4-84f4-eec72fa97496 /lustre/mgt/mgt -v
```

### 2.2.5. 部署MDS服务
#### 2.2.5.1. 节点node1
**创建mdt**
```bash
mkfs.lustre --mdt \
--fsname fs00 \
--index 0 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--servicenode 192.168.3.12@tcp \
--backfstype=ldiskfs \
--reformat /dev/sdc
```
如果mgs服务有多个，必须要同时指定多个mgsnode，而且第一个mgsnode必须是primary mgs。另外对于每一个lustre文件系统，mdt index序号必须从0开始，0代表整个文件系统的根目录。

**启动mds服务**
```bash
mkdir -p /lustre/mdt/mdt0
mount -t lustre -U e7a02748-62a3-4a3d-9c16-77ddd2df3227 /lustre/mdt/mdt0 -v
```

#### 2.2.5.2. 节点node2
**创建mdt**
```bash
mkfs.lustre --mdt \
--fsname fs00 \
--index 1 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.12@tcp \
--servicenode 192.168.3.11@tcp \
--backfstype=ldiskfs \
--reformat /dev/sdd
```
如果mgs服务有多个，必须要同时指定多个mgsnode，而且第一个mgsnode必须是primary mgs。另外对于每一个lustre文件系统，mdt index序号必须从0开始，0代表整个文件系统的根目录。

**启动mds服务**
```bash
mkdir -p /lustre/mdt/mdt1
mount -t lustre -U 51baa8b7-a946-4b48-b1c4-4dec39ccea31 /lustre/mdt/mdt1 -v
```

### 2.2.6. 部署OSS服务
#### 2.2.6.1. 节点node3
**创建ost**
```bash
mkfs.lustre --ost \
--fsname fs00 \
--index 0 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.13@tcp \
--servicenode 192.168.3.14@tcp \
--backfstype=ldiskfs \
--reformat /dev/sde
```

**启动oss服务**
```bash
mkdir -p /lustre/ost/ost0
mount -t lustre -U d87d0bd1-79aa-4ba5-a34a-a60afa3fe0ee /lustre/ost/ost0 -v
```

#### 2.2.6.2. 节点node4
**创建ost**
```bash
mkfs.lustre --ost \
--fsname fs00 \
--index 1 \
--mgsnode 192.168.3.11@tcp \
--mgsnode 192.168.3.12@tcp \
--servicenode 192.168.3.14@tcp \
--servicenode 192.168.3.13@tcp \
--backfstype=ldiskfs \
--reformat /dev/sdf
```

**启动oss服务**
```bash
mkdir -p /lustre/ost/ost1
mount -t lustre -U b561560f-2991-40f2-80c1-028e8285aefe /lustre/ost/ost1 -v
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
modprobe -v lustre
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