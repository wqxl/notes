# 1. 虚拟机管理
### 1.1. 显示所有的虚拟机
```bash
vboxmanage list vms
```

### 1.2. 显示所有正在运行的虚拟机
```bash
vboxmanage list runningvms
```

### 1.3. 启动一个虚拟机
```bash
vboxmanage startvm b9901b60-e1d2-4b6b-a38c-a9bde7af9c1d --type=headless
```

### 1.4. 关闭一个虚拟机
```bash
vboxmanage controlvm b9901b60-e1d2-4b6b-a38c-a9bde7af9c1d poweroff
```

### 1.5. 为虚拟机添加一个磁盘
```bash
vboxmanage storageattach ab74d222-a713-4c81-89f8-2e9b0d31ad3f --storagectl SATA --port 1 --device 0 --type hdd --medium ./centos-8.4-3.13-disk1.vdi
```

&nbsp;
# 2. 磁盘管理
### 2.1. 显示所有的磁盘
```bash
vboxmanage list hdds
```

### 2.2. 创建一个磁盘
```bash
vboxmanage createmedium --filename /vboxharddisk/centos-8.4-3.13/centos-8.4-3.13-disk1.vdi --size 1024 --format VDI
```

### 2.3. 删除一个磁盘
```bash
vboxmanage closemedium disk /vboxharddisk/centos-8.4-3.12/centos-8.4-3.12-disk1.vdi --delete
```

&nbsp;
# 3. 快照管理
### 3.1. 显示某个虚拟机所有快照
```bash
vboxmanage snapshot ab74d222-a713-4c81-89f8-2e9b0d31ad3f list
```

### 3.2. 创建一个快照
```bash
vboxmanage snapshot ab74d222-a713-4c81-89f8-2e9b0d31ad3f take raid1
```

### 3.3. 删除一个快照
```bash
vboxmanage snapshot ab74d222-a713-4c81-89f8-2e9b0d31ad3f delete d074a9aa-f910-48a1-ab97-f41eb411188c
```

### 3.4. 恢复一个快照
```bash
vboxmanage snapshot ab74d222-a713-4c81-89f8-2e9b0d31ad3f restore f25f2972-3631-4472-8de9-c48f798f2525
```