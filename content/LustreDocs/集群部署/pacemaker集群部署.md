> 系统版本：Centos 8.4.2150  
> Pacemaker版本：2.0.5  

&nbsp;
# 1. 安装软件
### 1.1. 软件安装
```bash
yum install pacemaker pcs psmisc fence-agents-ipmilan
```
也可以使用`rpm --reinstall --replacefiles -iUvh xxxx.rpm`命令离线安装。

### 1.2. 启动pcsd服务
```bash
systemctl start pcsd.service
systemctl enable pcsd.service
```
注：所有的pacemaker集群节点都需要执行以上操作。

&nbsp;
# 2. 用户配置
### 2.1. 设置用户密码
在安装pacemaker相关软件时，会自动创建一个名为hacluster而且没有密码的用户，但是在使用pcs相关命令时，需要提供hacluster的密码，因此需要为hacluster设置密码。
```bash
passwd hacluster
```
注：所有的pacemaker节点都需要执行该操作。

### 2.2. 用户授权
```bash
pcs host auth node0 node1 node2
```
注：该命令只需要在pacemaker一个节点上执行一次即可。

&nbsp;
# 3. 集群部署
### 3.1. 创建集群
```bash
pcs cluster setup mycluster node0 node1 node2
```
`mycluster`是集群的名字。

### 3.2. 启动集群
```bash
pcs cluster start --all
```
启动集群时会同时启动pacemaker.service和corosync.service，可以通过systemctl查看服务状态。

### 3.3. 设置pacemaker服务开机自启动
```bash
systemctl enable pacemaker.service
```

### 3.4. 设置corosync服务开机自启动
```bash
systemctl enable corosync.service
```

### 3.5. 查看集群状态
```bash
pcs status
```

&nbsp;
# 4. 集群配置
### 4.1. 设置stonith-enabled
fencing在集群中起到保护数据的作用，它是通过两种途径：切断电源和禁止访问共享存储。默认情况下Fencing功能是启用的,如果需要关闭，行以下命令：
```bash
pcs property set stonith-enabled=false
```

### 4.2. 设置resource-stickiness
在大多数情况下，当pacemaker集群中某个节点恢复正常，应阻止资源迁移到该节点上。资源迁移需要耗费大量时间。尤其是针对于非常复杂的服务。因此有必要阻止健康资源在集群节点中移动。pacemaker提供了resource-stickiness属性用来设置资源与节点的粘合度。数值越高，粘合度就越高，该资源就不会自动迁移到其他节点上。
```bash
pcs resource defaults resource-stickiness=100
```

### 4.3. 设置no_quorum_policy
`no_quorum_policy`属性用于定义当集群出现脑裂并且无法仲裁时，pacemaker集群应该如何运作。对于两节点的pacemaker而言，应该设置为ignore，表示集群继续正常运行。对于大于两节点的集群，应该设置为stop。
```bash
pcs property set no-quorum-policy=stop
```