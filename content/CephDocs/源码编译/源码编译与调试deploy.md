> Ceph版本：14.2.22  
> 系统版本：ubuntu 18.04  

&nbsp;
ceph-deploy默认使用官方发布的ceph安装包来搭建集群，如果想要调试ceph集群，需要安装对应的dbg包，或者使用本地编译出来的二进制文件和动态库进行相应的替换。对于后者而言，要保证源码必须和二进制文件以及动态库文件在同一台机器上。本文着重讲解通过不安装dbg包来调试使用官方发布的ceph包搭建的集群。

&nbsp;
&nbsp;
# 1. 安装依赖
ceph源码是通过`install-deps.sh`脚本安装编译需要的依赖。

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
如果想要调试Ceph源码，需要设置编译源码模式为debug模式，默认编译模式为release模式，该模式是不能调试源码。向 ceph/CMakeList 文件的 set(VERSION 14.2.22) 后追加以下内容。
```bash
set(CMAKE_BUILD_TYPE "Debug")
set(CMAKE_CXX_FLAGS_DEBUG "-O0 -Wall -g")
set(CMAKE_CXX_FLAGS "-O0 -Wall -g")
set(CMAKE_C_FLAGS "-O0 -Wall -g ")
```

### 2.3. 构建build目录
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
集群部署既可以采用离线部署也可以采用在线部署方式。为了加快部署进程，本文采用在线部署方式，详细部署可以参考[容灾集群部署](../集群部署/容灾集群部署.md)。另外，本文需要调试文件系统方面的功能，因此需要搭建好文件系统，详细部署可以参考[文件存储系统部署](../集群部署/文件存储系统部署.md)。


&nbsp;
&nbsp;
# 4. 调试
### 4.1. 替换二进制文件和动态库文件
默认情况下，官方发布版本是不可以调试，需要使用build/bin和/build/lib编译出来的关于mds的文件替系统默认安装的。系统默认把ceph相关的二进制文件安装到/usr/bin，把ceph依赖的动态库安装到/usr/lib中。本文需要调试文件系统mds管理元数据功能，只需要替换ceph-mds二进制文件。

### 4.2. 查看mds进程号
```bash
ps -e| grep ceph-mds
-----------------------------------------------------------------------------------------------------------------------
15201 ?		00:29:32 ceph-mds
```
从结果可以看到mds进程号为 `15201`。

### 4.3. GDB调试
**进入gdb模式**  
gdb调试需要以管理员权限，本文默认是root用户，所以直接输入gdb即可进入gdb模式。
```bash
gdb
-----------------------------------------------------------------------------------------------------------------------
GNU gdb (Ubuntu 7 .11.1- 0ubuntu1~16.5) 7.11.1
Copyright (C) 2016 Free Software Foundation, Inc .
L icense GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law. Type " show copying"
and” show warranty" for details.
This GDB was configured as "x86_64-1inux-gnu".
Type "show conf iguration" for configuration details.
For bug reporting instructions, pLease see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word".
(gdb)
```

**attach mds进程**  
```bash
(gdb) attach 15201
-----------------------------------------------------------------------------------------------------------------------
Attaching to process 15201
[New LWP 15203]
[New LWP 15204]
[New LWP 15205]
[New LWP 15206]
[New LWP 15211]
[New LWP 15212]
[New LWP 15213]
[New LWP 15214]
[New LWP 15215]
[New LWP 15216]
[New LWP 15217]
[New LWP 15218]
[New LWP 15219]
[New LWP 15220]
[New LWP 15223]
[New LWP 15224]
[New LWP 15225]
[New LWP 15226]
[New LWP 15227]
[New LWP 15228]
[New LWP 15229]
[New LWP 15231]
[Thread debugging using libthread_db enabled]
Using host libthread db library "/1ib/x86_64-linux-gnu/libthread_db.so.1" .
pthread_cond_wait@aGLIBC_2.3.2 () at ../sysdeps/unix/sysv/linux/x86_64/pthread_cond_wait.S:185
185		../sysdeps/unix/sysv/linux/x86_64/pthread_cond_wait.S: No such file or directory.
(gdb)
```

**设置断点**  
因为需要调试新建目录时，mds对元数据的管理，因此断点设置在Server::handle_client_mkdir起始处。
```bash
(gdb) b Server.cc:5913
Breakpoint 1 at 0x56086dc98df6: file /work/ceph-14.2.22/src/mds/Server.cc, line 5913.
```

设置完断点后，执行continue。
```bash
(gdb) c
Continuing.
```

**测试**  
在挂载出来的文件系统目录下，新建一个目录，代码会跳到之前设置的断点处。
```bash
Thread 9 "ms_dispatch" hit Breakpoint 1, Server::handle_client_mkdir (this=0x560870f16dc0, mdr=...)
at /work/ceph-14.2.22/src/mds/Server.cc:5914 
5914	{
(gdb) n
5915	const MClientRequest::const_ref &req = mdr->client_request;
```
从上面结果可以看到，新建一个目录时，函数停在代码的5914行，现在就可以使用gdb命令进行代码调试，和正常调试代码一样。
