> 系统版本：Centos 8.4.2150  
> 内核版本：4.18.0-305.3.1.el8.x86_64  
> lustre版本：2.15.3  
> backfstype：zfs  

&nbsp;
# 1. Primary Node1
### 1.1. 关闭mds服务
```bash
umount /lustre/mdt/mdt0000
```

### 1.2. 导出mdtpool
```bash
zpool export mdtpool0000
```

### 1.3. 检查mdtpool是否已经被导出
```bash
zpool list
```
如果在列出的pool中没有mdtpool-0，说明已经正确从Primary Node1中导出mdtpool-0。  
以上是模拟Primary Node1中mgs服务由于某种原因导致其无法正常工作。如果Primary Node1机器出现故障，比如出现断电，可以直接忽略以上步骤，直接按照下面步骤执行。需要注意的是mgt必须是共享盘。

&nbsp;
&nbsp;
# 2. Primary Node2
### 2.1. 导入mdtpool
```bash
zpool import -o cachefile=none mdtpool0000
```
注：mdtpool-0只能同时被一个host导入。

### 2.2. 检查mdtpool是否已经被导入
```bash
zpool list
```
如果在列出的pool中出现mdtpool-0，说明已经正确导入mdtpool-0。

### 2.3. 启动mds服务
```bash
mkdir -o /lustre/mdt/mdt0000
mount -t lustre mdtpool0000/mdt0000 /lustre/mdt/mdt0000 -v
```
