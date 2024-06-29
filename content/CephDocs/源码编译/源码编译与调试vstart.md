> Ceph版本：14.2.22  
> 系统版本：ubuntu 18.04  

&nbsp;
# 1. 安装依赖
### 1.1. 设置pypi镜像源
脚本会安装pypi库，默认url下载很慢，需要设置pypi库镜像源。创建 ~/.pip/pip.conf 文件，并追加以下内容。
```bash
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host=mirrors.aliyun.com
```

### 1.2. 安装基础依赖
```bash
./install-deps.sh
```

### 1.3. 安装其他依赖
编译源码过程中会遇到很多函数用到zstd库，默认情况下ubuntu18.04只安装了libzstd1，但没有用，需要安装 libzstd1-dev。
```bash
apt install libzstd1-dev
```

&nbsp;
&nbsp;
# 2. 编译
### 2.1. 获取源码
本文采用从阿里云镜像源上直接下载[https://mirrors.aliyun.com/ceph/debian-nautilus/pool/main/c/ceph/ceph_14.2.22.orig.tar.gz]()，而不是从Github上拉代码。ceph源码包中包含了ceph整个项目的源码（包括使用的第三方源码），所以不用担心源码缺失问题，并且可以直接通过国内开源镜像站去下载，不用担心下载慢的问题。

### 2.2. 开启debug模式
如果想要调试Ceph源码，需要设置编译源码模式为debug模式，默认编译模式为release模式，该模式是不能调试源码。修改ceph/CMakeList文件，在`set(VERSION 14.2.22)`后追加以下内容。
```bash
set(CMAKE_BUILD_TYPE "Debug")
set(CMAKE_CXX_FLAGS_DEBUG "-O0 -Wall -g")
set(CMAKE_CXX_FLAGS "-O0 -Wall -g")
set(CMAKE_C_FLAGS "-O0 -Wall -g ")
```

### 2.3. 生成build目录
直接执行do_cmake脚本，该脚本会进行一系列检测，包括源码是不是完整，依赖是不是都安装了等等。如果出现问题，构建出的build目录是不完整的，最直接的影响是无法生成makefile文件，导致无法编译。
```bash
./do_cmake.sh
```

### 2.4. 编译
使用make编译必须要到ceph/build目录下执行，ceph源码可以单独编译某一个模块，也可以全部编译。使用make可以指定多线程编译，提高编译速度，但要合理分配线程数，建议使用4线程编译即可。
```bash
#方式1：全部编译
make all -j4
#方式2：单独编译osd某块
make ceph-osd -j4
#查看所有模块
make help
```
源码编译会生成很多库文件和二进制文件，分别放在ceph/build/lib和ceph/build/bin目录下。


&nbsp;
&nbsp;
# 3. 部署
Cpeh源码提供了vstart.sh脚本来部署开发集群，该脚本会利用本地ip和不同端口来配置MON、MGR、OSD等。vstart.sh脚本需要在build目录下运行。执行`../src/vstart.sh -h`可以查看vstar.sh使用说明。
```bash
../src/vstart.sh -h
-----------------------------------------------------------------------------------------------------------------------
ex: MON=3 OSD=1 MDS=1 MGR=1 RGW=1 ../src/vstart.sh -n -d
options:
    -d, --debug
    -s, --standby_mds: Generate standby-replay MDS for each active
    -l, --localhost: use localhost instead of hostname
    -i <ip>: bind to specific ip
    -n, --new
    -N, --not-new: reuse existing cluster config (default)
    --valgrind[_{osd,mds,mon,rgw}] 'toolname args...'
    --nodaemon: use ceph-run as wrapper for mon/osd/mds
    --smallmds: limit mds cache size
    -m ip:port		specify monitor address
    -k keep old configuration files
    -x enable cephx (on by default)
    -X disable cephx
    -g --gssapi enable Kerberos/GSSApi authentication
    -G disable Kerberos/GSSApi authentication
    --hitset <pool> <hit_set_type>: enable hitset tracking
    -e : create an erasure pool
    -o config		 add extra config parameters to all sections
    --rgw_port specify ceph rgw http listen port
    --rgw_frontend specify the rgw frontend configuration
    --rgw_compression specify the rgw compression plugin
    -b, --bluestore use bluestore as the osd objectstore backend (default)
    -f, --filestore use filestore as the osd objectstore backend
    -K, --kstore use kstore as the osd objectstore backend
    --memstore use memstore as the osd objectstore backend
    --cache <pool>: enable cache tiering on pool
    --short: short object names only; necessary for ext4 dev
    --nolockdep disable lockdep
    --multimds <count> allow multimds with maximum active count
    --without-dashboard: do not run using mgr dashboard
    --bluestore-spdk <vendor>:<device>: enable SPDK and specify the PCI-ID of the NVME device
    --msgr1: use msgr1 only
    --msgr2: use msgr2 only
    --msgr21: use msgr2 and msgr1
```

### 3.1. 新建集群
```bash
MON=1 OSD=6 MDS=0 MGR=1 RGW=0 ../src/vstart.sh -d -n -x --without-dashboard
```

### 3.2. 重启集群或服务
目前ceph 14.2.22的vstart.sh脚本还做不到可以单独重启某一个服务，如果需要重启某一个服务，只需要去掉`-n`参数即可。
```bash
MON=1 OSD=6 MDS=0 MGR=1 RGW=0 ../src/vstart.sh -d -x --without-dashboard
```
上述命令是重启集群所有服务，该命令不会新建集群。

### 3.3. 停止集群或服务
ceph 14.2.22的源码中提供了stop.sh脚本用来停止集群中的服务，目前该脚本还做不到单独停止某一个服务，但是可以停止某一类服务，也可以停止所有的服务，具体用法如下：
```bash
../src/stop.sh [all] [mon] [mds] [osd] [rgw]
```
经过测试发现，对象存储、块存储都可以正常部署并调试，文件系统部署可以正常部署，但是无法挂载。如果想要调试所有功能，不建议采取这种方式。建议生成deb包，然后按正常ceph部署，最后替换对用的动态库和二进制可执行文件即可调试。

### 3.4. 查看集群状态
ceph 14.2.22版本的vstart.sh脚本并没有将ceph可执行文件添加到系统环境变量中，所有的ceph命令都必须在build目录下执行。切换到build目录下，执行以下命令，查看集群状态。
```bash
./bin/ceph -s
-----------------------------------------------------------------------------------------------------------------------
cluster:
  id: 88b11a21- 7dd1- 49d8. bb24-C 18821ff09ae
  health: HEALTH ok

services:
  mon: 1 daemons, quorum a (age 5m)
  mgr: x(active, since 5m)
  osd: 6 osds: 6 up (since 4m)，6 in (since 4m)
data:
  pools:
  pools, 0 pgs
  objects: 0 objects, 0 B
  usage: 12 G1B used, 594 GiB / 606 GiB avail
  pgs:
```

执行`./bin/ceph -s`后默认会在终端输出如下信息：
```bash
2023-07-31 03:52:04.851 7fcff7cc5700 -1 WARNING: all dangerous and experimental features are enabled.
2023-07-31 03:52:04.907 7fcff7cc5700 -1 WARNING: all dangerous and experimental features are enabled.
```
如果不想输出这些信息，将ceph.conf文件中的参数`enable experimental unrecoverable data corrupting features = *`屏蔽掉就可以了。

### 3.5. 部署ceph分级存储结构
本案例需要调试ceph分级存储功能，因此简单的搭建一个分层存储结构。为集群分配6个OSD，创建2个pool，cache pool和ec pool，每个pool分配了3个osd。具体部署参考[分层存储系统部署]()。

&nbsp;
&nbsp;
# 4. 调试
### 4.1. 查看PG-OSD映射关系
如果仔细阅读源码，会发现ceph分级存储主要是由主OSD进程来负责。如果不是主OSD，是无法调试到代码中的。所以需要查看分级存储中缓存池的PG映射关系。
切换到build目录下，执行以下命令
```bash
./bin/ceph pg ls-by-pool cache_pool
-----------------------------------------------------------------------------------------------------------------------
PG OBJECTS DEGRADED MISPLACED UNFOUND BYTES OMAP_BYTES* OMAP_KEYS* LOG STATE 		SINCE VERSION REPORTED UP 		   ACTING		SCRUB_STAMP 					DEEP_SCRUB_STAMP
5.0		0		0			0 		0	0			0			0	18 active+clean   22h 	323'18 323:76 [2, 4,0]p2   [2,4,0]p2	2021-09-25 16:55:28.572062		2021-09-24 11:30:14.717641
```
从结果可以看到PG5.0对应的主OSD为OSD.2。

### 4.2. 查看主OSD进程
```bash
ps -ef | grep ceph
-----------------------------------------------------------------------------------------------------------------------
admins 10961 19680 0 15:12 pts/0 00:00:00 grep --color=auto ceph
admins 18474     1 1 Sep24 ?     01:02:09 /home/admins/code/ceph/build/bin/ceph-mon -i a -C /home/admins/code/ceph/build/ceph.conf
admins 18582     1 1 Sep24 ?	 00:33:41 /home/admins/code/ceph/build/bin/ceph-mgr -i x -C /home/admins/code/ceph/build/ceph.conf
admins 18806     1 1 Sep24 ?	 00:41:15 /home/admins/code/ceph/build/bin/ceph-osd -i 1 -C /home/admins/code/ceph/build/ceph.conf
admins 19096     1 1 Sep24 ?	 00:41:06 /home/admins/code/ceph/build/bin/ceph-osd -i 3 -C /home/admins/code/ceph/build/ceph. conf
admins 19242     1 1 Sep24 ?	 00:40:37 /home/admins/code/ceph/build/bin/ceph-osd -i 4 -C /home/admins/code/ceph/build/ceph.conf
admins 19415     1 1 Sep24 ?	 00:41:00 /home/admins/code/ceph/build/bin/ceph-osd -i 5 -C /home/admins/code/ceph/build/ceph.conf
admins 20385     1 1 Sep24 ?	 00:39:47 /home/admins/code/ceph/build/bin/ceph-osd -i 0 -C /home/admins/code/ceph/build/ceph.conf
admins 22235     1 1 Sep24 ?	 00:40:24 /home/admins/code/ceph/build/bin/ceph-osd -i 2 -C /home/admins/code/ceph/build/ceph.conf
```
从结果可以看到，主OSD进程号为 `22235`。

### 4.3. GDB调试
**进入gdb模式**  
gdb调试需要以管理员权限，执行以下命令，进入gdb模式。
```bash
sudo gdb
-----------------------------------------------------------------------------------------------------------------------
[sudo] password for admins :
GNU gdb (Ubuntu 8.1. 1- Oubuntu1) 8.1.1
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law. Type "show copying"
and "show warranty" for details.
This GDB was configured as" x86 64- linux- gnu".
Type”show configuration" for configuration details.
For bug repor ting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word".
(gdb)
```

**attach osd2 进程**  
```bash
(gdb) attach 22235 
-----------------------------------------------------------------------------------------------------------------------
Attaching to process 22235
[New LWP 22237]
[New LWP 22238]
[New LWP 22239]
[New LWP 22248]
[New LWP 22249]
[New LWP 22250] 
[New LWP 22251]
[New LWP 22254]
[New LWP 22255]
[New LWP 22256]
[New LWP 22257]
[New LWP 22258]
[New LWP 22259]
[New LWP 22260]
[New LWP 22269]
[New LWP 22270] 
[New LWP 22271]
[Thread debugging using libthread db enabled]
Using host libthread db library "/lib/x86_64-linux-gnu/libthread db.so.1"
0x00007fd026a7dad3 in futex_ wait_ cancelable (private=<optimized out>, expected=0, futex_ word=0x55b3123d8910) at ../sysdeps/unix/sysv/Linux/futex-internal.h:8888	../sysdeps/unix/sysv/1inux/futex-internal.h: No such file or directory.
(gdb)
```

**设置断点**  
本例断电设置在PrimaryLogPG::do_op函数开始，设置完断点之后，执行continue。
```bash
(gdb) b PrimaryLogPG.cc:1952
Breakpoint 1 at 0x55b305d28af2: file /home/admins/code/ceph/src/osd/PrimaryLogPG.cc, line 1952.
(gdb ) c
Continuing.
```

**测试**  
向存储池中写入数据，测试结果如下。
```bash
[Switching to Thread 0x7fd0034cb700 (LWP 22364)]
Thread 57 "tp_osd_tp" hit Breakpoint 1, PrimaryLogPG::do_op (this=0x55b312519400, op=...)
at /home/admins/code/ceph/src/osd/PrimaryLogPG.CC:1952
1952		{
```
从上面结果可以看到，当写入数据时，函数停在代码的1952行，现在就可以使用gdb命令进行代码调试，和正常调试代码一样。但需要值得注意的一点是，由于ceph osd存在心跳机制，当调试某一个osd时，如果长时间没有走完该走的流程，该osd会被标记为down，就无法再继续调试。需要重新进入gdb模式。