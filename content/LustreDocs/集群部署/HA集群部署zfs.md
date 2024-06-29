> 系统版本：Centos 8.4.2150  
> 内核版本：4.18.0-305.3.1.el8.x86_64  
> lustre版本：2.15.3  
> Pacemaker版本：2.0.5  
> backfstype：zfs  

&nbsp;
本文默认pacemaker和lustre集群已经提前部署。

# 1. 集群规划
```bash
mgt      192.168.3.11     node1
mdt0     192.168.3.11     node1
mdt1     192.168.3.12     node2
ost0     192.168.3.13     node3
ost1     192.168.3.14     node4
```
node1与node2互为主备，node3与node4互为主备。

&nbsp;
&nbsp;
# 2. 安装软件
### 2.1. 安装zfs agent
```
cp /opt/ZFS /usr/lib/ocf/resource.d/heartbeat/
```

### 2.2. 查看zfs agent是否已经安装
```bash
pcs resource agents ocf:heartbeat
```
如果输出中有ZFS，说明已经安装。或者执行`pcs resource list ocf:heartbeat:ZFS`也可以查看。

&nbsp;
# 3. 创建resourse
### 3.1. 创建mgtpool资源
```bash
pcs resource create mgtpool ocf:heartbeat:ZFS pool="mgtpool"
```

### 3.2. 创建mgt资源
```bash
pcs resource create mgt ocf:lustre:Lustre \
target="mgtpool/mgt" \
mountpoint="/lustre/mgt/mgt"
```

### 3.3. 创建mdt0pool资源
```bash
pcs resource create mdt0pool ocf:heartbeat:ZFS pool="mdt0pool"
```

### 3.4. 创建mdt0资源
```bash
pcs resource create mdt0 ocf:lustre:Lustre \
target="mdt0pool/mdt0" \
mountpoint="/lustre/mdt/mdt0"
```

### 3.5. 创建mdt1pool资源
```bash
pcs resource create mdt1pool ocf:heartbeat:ZFS pool="mdt1pool"
```

### 3.6. 创建mdt1资源
```bash
pcs resource create mdt1 ocf:lustre:Lustre \
target="mdt1pool/mdt1" \
mountpoint="/lustre/mdt/mdt1"
```

### 3.7. 创建ost0pool资源
```bash
pcs resource create ost0pool ocf:heartbeat:ZFS pool="ost0pool"
```

### 3.8. 创建ost0资源
```bash
pcs resource create ost0 ocf:lustre:Lustre \
target="ost0pool/ost0" \
mountpoint="/lustre/ost/ost0"
```

### 3.9. 创建ost1pool资源
```bash
pcs resource create ost1pool ocf:heartbeat:ZFS pool="ost1pool"
```

### 3.10. 创建ost1资源
```bash
pcs resource create ost1 ocf:lustre:Lustre \
target="ost1pool/ost1" \
mountpoint="/lustre/ost/ost1"
```

&nbsp;
# 4. 配置resourse规则
以下各项规则根据需要配置。

### 4.1. 添加location规则
```bash
pcs constraint location mgtpool prefers node1=INFINITY node2=INFINITY node3=-INFINITY node4=-INFINITY
pcs constraint location mdt0pool prefers node1=INFINITY node2=INFINITY node3=-INFINITY node4=-INFINITY
pcs constraint location mdt1pool prefers node1=INFINITY node2=INFINITY node3=-INFINITY node4=-INFINITY
pcs constraint location ost0pool prefers node1=-INFINITY node2=-INFINITY node3=INFINITY node4=INFINITY
pcs constraint location ost1pool prefers node1=-INFINITY node2=-INFINITY node3=INFINITY node4=INFINITY
```
以上配置的location规则表示：mgt资源在node1和node2之间随机选择（pacmaker默认是随机选择节点），并且绝不选择node3和node4。数值相等时是随机选择，数值不相等时，优先选择数值大的节点。

### 4.2. 添加colocation规则
```bash
pcs constraint colocation add mgt with mgtpool INFINITY
pcs constraint colocation add mdt0 with mdt0pool INFINITY
pcs constraint colocation add mdt1 with mdt1pool INFINITY
pcs constraint colocation add ost0 with ost0pool INFINITY
pcs constraint colocation add ost1 with ost1pool INFINITY
```
上述配置的colocation规则表示：mgt mgtpool资源必须在同一个节点上启动，其他类推。如果想要让mgt mgtpool资源绝对不能在同一个节点上启动，只要将`INFINITY`变成`-INFINITY`。

### 4.3. 添加ordering规则
```bash
pcs constraint order set mgtpool mgt sequential=true require-all=true action=start \
                     set ost0pool ost1pool sequential=false require-all=false action=start \
                     set ost0 ost1 sequential=false require-all=false action=start \
                     set mdt0pool mdt1pool sequential=false require-all=false action=start \
                     set mdt0 mdt1 sequential=false require-all=false action=start
```
上述配置的ordering规则表示：mgtpool mgt资源是按照顺序启动并且必须全部成功启动，然后启动ost0pool ost1pool资源，启动顺序是无序且无需全部成功启动，然后启动ost0 ost1资源，启动顺序是无序且无需全部成功启动，然后启动mdt0pool mdt1pool资源，启动顺序是无序且无需全部成功启动，最后启动mdt0 mdt1资源，启动顺序是无序且无需全部成功启动。该规则是将多组ordering set组合成一个ordering，set之间成为了相互依赖关系。如果想要将set独立，只要单独执行`pcs constraint order set`即可。

&nbsp;
# 5. 配置fence
当pacemaker无法停掉某个服务时，可以通过fence强制将该服务所在的机器关机，然后将该服务在其他机器上再次启动。
### 5.1. 开启stonith-enabled
```bash
pcs property set stonith-enabled=true
```
### 5.2. 创建fence0 resource
```bash
pcs stonith create fence0 fence_ipmilan \
        ip="192.168.19.64" \
        username="admin" \
        password="admin" \
        pcmk_host_list = "node1"
        pcmk_host_check = "static-list"
```
`fence0`为resource name，`fence_ipmilan`为fence agent名字，fence agent的名字可以通过`pcs stonith list`命令查看，`ip`为ipmi地址，`pcmk_host_list`为ipmi管理的服务器地址。其余参数都是`fence_ipmilan`所支持的options，可以通过`pcs stonith describe fence_ipmilan`命令查看`fence_ipmilan`所有的options。

### 5.3. 创建fence1 resource
```bash
pcs stonith create fence1 fence_ipmilan \
        ip="192.168.19.64" \
        username="admin" \
        password="admin" \
        pcmk_host_list = "node2"
        pcmk_host_check = "static-list"
```

### 5.4. 创建fence2 resource
```bash
pcs stonith create fence2 fence_ipmilan \
        ip="192.168.19.64" \
        username="admin" \
        password="admin" \
        pcmk_host_list = "node3"
        pcmk_host_check = "static-list"
```

### 5.5. 创建fence3 resource
```bash
pcs stonith create fence3 fence_ipmilan \
        ip="192.168.19.65" \
        username="admin" \
        password="admin" \
        pcmk_host_list = "node4"
        pcmk_host_check = "static-list"
```