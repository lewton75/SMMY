# 环境搭建

本人使用电脑的是 64 位的 **Macbook pro**，然后使用虚拟机软件 - VMware Fusion。在 Fusion 中安装 `ubuntu-14.04.5-desktop-desktop-i386.iso` 系统。

> 踩过的坑：之前我是使用的 VirtualBox 软件来跑 Ubuntu 客户操作系统，不知道是什么原因，使用 Qemu 成功启动实验构建的系统后，整个 VirtualBox 都会卡住。
>
> 一开始我以为因为在虚拟机 VB 中跑虚拟机 QEMU，且又使用了 NFS 协议（*QEMU 中运行的 Linux 内核，通过 NFS 协议将远端的特定目录挂载到自己的目录中，远端指的是 VB 中运行的 Ubuntu 系统*），所以才导致 VB 卡住了。然后试了下不采用 NFS 协议挂载 Linux 内核根文件系统的方法，然而还是出现同样的问题。
> 
> 所以**坑**是VirtualBox。

## Jin Yang 的环境

需要编译安装的软件如下：

1. [Qemu-1.4.2](​Qemu-1.4.2)

2. Linux 内核——[Linux-3.4.48](https://www.kernel.org/pub/linux/kernel/v3.0/linux-3.4.48.tar.bz2)

3. 静态编译的 [Busybox-1.21.0](http://www.busybox.net/downloads/busybox-1.21.0.tar.bz2)

### 1. 安装 QEMU

使用 `git` 将 `qemu-can` 工程克隆到本地，[https://github.com/Jin-Yang/QEMU-1.4.2]()：

```bash
 git clone https://github.com/Jin-Yang/QEMU-1.4.2.git QEMU-1.4.2
```

然后使用下面的命令配置 QEMU。

```bash
./configure --prefix=/opt/qemu --target-list="i386-softmmu" --enable-sdl
```

`./configure ...` 首先会报错：

```bash
Error: zlib check failed
Make sure to have the zlib libs and headers installed.
```

解决方法：

```bash
sudo apt-get install zlib1g-dev
```

继续执行 `./configure ...`，又出现了下面的错误：

```bash
ERROR
ERROR: User requested feature sdl
ERROR: configure was not able to find it
ERROR
```

解决方法，安装 `sdl`：

```bash
sudo apt-get install libsdl1.2-dev
```

`./configure ...` 配置成功，接着进行 `make` 编译过程，编译过程失败，需要安装 `autoconf`，然后继续 `make`：

```bash
sudo apt install autoconf
```

继续make 过程中出现如下错误：

```bash
...
configure.ac:75: error: possibly undefined macro: AC_PROG_LIBTOOL
...
```

解决方法，安装`libtool`后，继续 `make`：

```bash
sudo apt install libtool
```

`make` 最后成功后，运行 `sudo make install`，最后再目录 `/opt/qemu/bin` 中会出现我们需要的 `qemu-system-i386` 可执行文件。

> 自己在安装过程中主要参考了此[链接](https://theintobooks.wordpress.com/2016/08/15/installing-qemu-on-ubuntu-16-04/)。

### 2. 编译 Linux 内核

[链接](https://devel.rtems.org/wiki/Developer/Simulators/QEMU/CANEmulation#BuildingLinuxEnvironment)的参考中有一个生成最简内核镜像的步骤，我弄过一次，比较繁琐。可以直接采用下面的暴力方式：

```
# 解压内核
tar -jxvf linux-3.4.48.tar.bz2
# 配置内核
make i386_defconfig
# 生成内核
make bzImage
```

最后成功生成需要的文件：

```bash
find -name bzImage
./arch/x86/boot/bzImage
./arch/i386/boot/bzImage
```

### 3. 结合 BusyBox 制作 Linux 根目录文件系统

**静态编译 BusyBox** 

安装 busybox 的操作命令如下，其中在执行 `make menuconfig` 时，需要 `libncurses5-dev` 库。

```bash
tar -jxvf busybox-1.20.0.tar.bz2
cd busybox-1.20.0
make defconfig
make menuconfig  # '''NOTE:''' ''Busybox Settings -> Build options -> Build Busybox as a static binary''
make
make install CONFIG_PREFIX=~/qemu/rootfs
```

上面的操作成功后，我们需要的 BusyBox 的可执行文件全部放在了 `~/qemu/rootfs` 中了。

接下来，我们还需要添加必要的目录和启动脚本，来满足根文件系统的结构要求。

由于对 Linux 启动过程不是十分了解，同时对 Linux 内核启动完成后，挂载根文件系统的详细步骤也不是很清楚。所以直接使用作者 Jin Yang 的实验环境了，链接为 [https://github.com/Jin-Yang/LINUX-QEMU.git](https://github.com/Jin-Yang/LINUX-QEMU.git)。**本机中的存放地址为：`/home/lewton/work/author`，其根文件系统的路径为：`/home/lewton/work/author/rootfs`**

> 搭建简单的，将根目录文件系统制作成镜像文件的方式，可以参看下面的链接：
> 
> [How to Build A Custom Linux Kernel For Qemu](http://weng-blog.com/2015/05/Build-Linux-kernel-rootfs-from-scratch/)
> 
> [Building linux system (kernel + root file system) from scratch](http://weng-blog.com/2015/05/Build-Linux-kernel-rootfs-from-scratch/)

## 测试并运行环境

完成上面的步骤后，我们已经有了实验环境。因为 Linux 内核的根文件系统使用的是 `NFS` 协议，即使我们克隆了作者的实验环境，我们也还需要进行下面的操作，才能正确地将环境运行起来。

**(1)** 在 VMware Fusion 的 Ubuntu 操作系统中安装 NFS 工具，然后修改配置文件，最后启动 NFS 服务，详细的步骤如下，参考[Linux NFS Root under QEMU ARM emulator](https://balau82.wordpress.com/2010/04/27/linux-nfs-root-under-qemu-arm-emulator/)：)：

```bash
sudo apt-get install nfs-kernel-server
```

安装成功后，修改配置文件 `/etc/exports`，添加下面的一行信息：

```vim
/home/lewton/work/author/rootfs *(rw,sync,no_subtree_check,all_squash,insecure,anonuid=1000,anongid=1000)
```

然后执行下面的命令：

```bash
sudo exportfs -av
```

最后重新启动 NFS 服务：

```bash
sudo service nfs-kernel-server restart 
```

**(2)** 修改根目录文件系统中用于 NFS 通信的地址信息。

需要注意的是，最好使用 VMware Fusion 的**仅主机模式**的网络配置，下面的操作是在该模式下进行的。

首先修改 `qemu.sh` 文件，被修改的内容修改之后，如下所示：

```vim
$PREFIX/qemu-system-i386 -s -m 1024 -kernel bzImage \
    -append "notsc clocksource=acpi_pm root=/dev/nfs nfsroot=192.168.15.128:$WORKSPACE/rootfs rw \  
    ip=192.168.15.110:192.168.15.128:192.168.15.1:255.255.255.0::eth0:auto \
    init=/sbin/init" \
    -net nic,model=e1000,vlan=0,macaddr=00:cd:ef:00:02:01 \
    -net tap,vlan=0,ifname=tap0,script=$WORKSPACE/ifup-qemu \
    -serial none \
    -chardev vc,id=UDPserial \
    -device pci-serial,chardev=UDPserial \
    -chardev can,id=sja1000,port=vcan0 \
    -device pci-can,chardev=sja1000
```

其中有关 `NFS` 地址的说明，参见链接[Mounting the root filesystem via NFS (nfsroot)](https://www.kernel.org/doc/Documentation/filesystems/nfs/nfsroot.txt)。简单说明一下，上面 `ip` 配置后面跟着的 4 个 `ip` 地址，它们分别是：

* `192.168.15.110` 是 `QEMU` 上面加载的Linux 操作系统的 `ip` 地址。
* `192.168.15.128` 是 `VMware Fusion` 上面加载的 Ubuntu 操作系统的 `ip` 地址，即 `NFS` 服务器的地址。
* `192.168.15.1` 是 网关地址；由于 `VMware Fusion` 配置的网络模式为仅主机模式，所以该地址也是主机 mac 虚拟网卡的地址。
* `255.255.255.0` 一看就知道是子网掩码了。

接着修改 `ifup-qemu` 文件，修改后的内容如下：

```vim
#!/bin/sh
ifconfig $1 192.168.15.128
```

最后，还需要修改文件 `rootfs/etc/init.d/rcS` 中的内容，这里是非常容易忽略的一个地方。需要修改的内容也是和地址相关，写改完后，其内容如下：

```vim
/sbin/ifconfig lo 127.0.0.1 netmask 255.0.0.0
/sbin/ifconfig eth0 192.168.15.110 netmask 255.255.255.0
/sbin/route add default gw 192.168.15.1
```

至此，所有的前提条件已经满足了，但是要想成功运行 `qemu.sh`，还需要在 Fusion 的 Ubuntu 系统中添加虚拟设备 `SocketCAN`：

```bash
sudo insmod /lib/modules/`uname -r`/kernel/drivers/net/can/vcan.ko
sudo ip link add type vcan
sudo ip link set vcan0 up
```

执行 `qemu.sh` 脚本，顺利启动 `Linux` 操作系统：

```bash
sudo ./qemu.sh
```
