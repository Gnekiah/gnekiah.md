---
title: Femu 源码简析与测试环境配置
date: 2018-12-05 11:19:59
tags:
---


Femu 来自于 fast-18 上发布的一篇论文[The CASE of FEMU: Cheap, Accurate, Scalable and Extensible Flash Emulator][1][1]。首先 Femu 基于 Qemu 虚拟机实现的，在 Qemu 虚拟机中，对模拟 nvme 的模块进行了部分扩展，以支持更加高级别的针对 Lightnvm 的仿真功能。与原生的 Qemu-nvme 相比，Femu 的扩展主要集中在延迟仿真上。


----------


## Qemu-nvme 简介

Qemu 与宿主机/客户机系统的示意图如下，Qemu 是运行在宿主机之上的一个应用程序，在这个应用程序中，虚拟出一个硬件平台，例如 x86 架构或 arm 架构。在这之上再运行客户机系统，也就是跑在虚拟机上的操作系统。Qemu 模拟硬件平台包含很多东西，其中就有我们关注的块设备模拟，即 NVMe 设备。
!!!
<center>
!!!
![20181130202324.jpg][2]
!!!
</center>
!!!

Nvme 模块需要实现下面的函数——read 和 write：
``` C
static const MemoryRegionOps nvme_mmio_ops = {
    .read = nvme_mmio_read,
    .write = nvme_mmio_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .impl = {
        .min_access_size = 2,
        .max_access_size = 8,
```
这两个函数是对于 Qemu 而言的块设备模块的入口（在这里是扩展了 femu-oc 的 nvme 模块）。其中 read 负责读取寄存器的值，write 则负责进行具体的请求处理细节并进行写寄存器的值。写寄存器函数 nvme_mmio_write() 的功能大致如下图所示，包含两个类型的命令操作——Admin IO 和普通 IO。
!!!
<center>
!!!
![20181130202325.jpg][3]
!!!
</center>
!!!
Admin IO 的操作与 Lightnvm 驱动相关层的操作一一对应，分别是获取设备 Geometry 信息、获取映射表、获取坏块表以及更新坏块表。通用 IO 操作也与 Lightnvm 驱动相关层一一对应，包括了读、写、擦三个操作。下面介绍这两种类型的延迟仿真方式。


----------


## Admin IO 的延迟仿真

Femu 基于 Qemu-nvme 所做的修改主要就是一个部分——延迟仿真。Admin IO 的延迟仿真示意图如下图所示，首先从 Submission Queue 中取出一条请求，然后执行这条请求，执行完毕后，将请求插入到一个处理完成的队列中，然后更新 CQ（Completion Queue）定时器时间（当前时间+500纳秒），之后更新 SQ（Submission Queue）定时器（当前时间+10000纳秒）。CQ 定时器触发后，将执行结果插入到 Completion Queue 后，按下 CQ Doorbell。另一边 SQ 定时器触发后，将进行下一个请求的处理任务。
!!!
<center>
!!!
![20181130202326.jpg][4]
!!!
</center>
!!!


----------


## General IO 的延迟仿真

General IO 的延迟仿真示意图如下所示，与 Admin IO 相似，首先从 Submission Queue 中取出一条请求，然后执行这条请求，执行完毕后，与 Admin IO 不同的一点在于：General IO 的执行流程包含了读、写某个芯片、数据传输等操作的延迟，因此必须要计算执行操作所需要的时间。计算出操作所需时间后，将请求插入到一个处理完成的队列中，然后更新 CQ（Completion Queue）定时器时间（当前时间 +500 纳秒），之后更新 SQ（Submission Queue）定时器（当前时间 + 当前 IO 操作所需要的时间）。CQ 定时器触发后，将执行结果插入到 Completion Queue 后，按下 CQ Doorbell。另一边 SQ 定时器触发后，将进行下一个请求的处理任务。
!!!
<center>
!!!
![20181130202327.jpg][5]
!!!
</center>
!!!


----------


## 本文所用的环境

Femu 的作者已将启动命令以脚本的形式保存，下面介绍 Femu 安装所需要的环境以及运行的步骤。

1. Python 2.7
2. Ubuntu 18.04
3. Linux 4.14

此外需要额外安装的包与软件已经集合到 femu/femu-scripts/pkgdep.sh 脚本，直接执行即可。


----------


## Femu 安装过程记录

1. git 拷贝一份 Femu 的源码
``` bash
git clone git@github.com:ucare-uchicago/femu.git
```
2. 进入 femu/build-femu，如果没有这个文件夹，可以自行在同级目录下面创建
3. 将 femu/femu-scripts 文件夹下面的所有文件复制到 build-femu 文件夹下。实际上不需要全部复制，在 femu/femu-scripts/femu-copy-scripts.sh 文件中制定了需要复制的文件，所以实际上只需要复制 femu-copy-scripts.sh 这个脚本到 build-femu 文件夹下，然后执行即可。另一种快捷的方式是直接：
``` bash
cp ../femu-scripts/*.sh ../build-femu
cp ../femu-scripts/ftk ../build-femu
cp ../femu-scripts/vssd1.conf ../build-femu
```
4. 运行 femu-compile.sh
5. 建议创建一个格式为 qcow2 的镜像文件（其他格式的镜像也可以，qcow2 是动态扩展空间。其他格式在启动 qemu 的时候改一下对应命令的格式就可），这个文件相当于我们的磁盘设备，创建好之后给这个镜像（磁盘）安装系统。
``` bash
qemu-img create –f qcow2 u14s.qcow2 20G
qemu-system-x86_64 –m 2G –enable-kvm u14s.qcow2 –cdrom ubuntu18.04-beta2-desktop-amd64.iso
```
6. 通过配置当前目录下面的 conf 文件可以配置 SSD 的不同参数，仿真延时可以配置不同的 channel 和更多的具体的参数；例如：
``` bash
PAGE_SIZE               4096
PAGE_NB                 256
SECTOR_SIZE             512
FLASH_NB                64
BLOCK_NB                16
PLANES_PER_FLASH        1
LOG_RAND_BLOCK_NB       0
LOG_SEQ_BLOCK_NB        0
REG_WRITE_DELAY         40000
CELL_PROGRAM_DELAY      800000
REG_READ_DELAY          60000
CELL_READ_DELAY         40000
BLOCK_ERASE_DELAY       3000000
CHANNEL_SWITCH_DELAY_R  16
CHANNEL_SWITCH_DELAY_W  33
IO_PARALLELISM          0
WRITE_BUFFER_FRAME_NB   2048
READ_BUFFER_FRAME_NB    2048
CACHE_IDX_SIZE          10
CHANNEL_NB              8
OVP                     0
GC_MODE                 2
```
7. 通过 qemu 启动的我们的系统时，可以通过 qemu 给系统加载对应的设备和驱动，并且配置不同的参数（每次启动系统都通过 ./qemu-img 来读取配置文件创建对应的设备镜像 raw），例如可以从脚本 femu/femu-scripts/run-whitebox.sh 中来看到，所以运行的时候只需要运行这个集合命令集的 shell 脚本
``` bash
./qemu-img create -f raw $NVMEIMGF $NVMEIMGSZ
```
8. 进入系统需要挂载设备，首先找到设备名字和位置，通过 lsblk 可以看到还未格式化和挂载的创建 nvme 设备
9. 我们可以把这个磁盘挂载到 /tmp/ 下。首先创建一个文件夹 mkdir ene_test，然后格式化设备为 ext4 格式，mkfs.ext4 /dev/nvme0n1，接下来将格式化的设备挂载到我们对应的文件夹 ene_nvme 下：
``` bash
mount /dev/nvme0n1 ene_nvme/
```
10. 到此可以通过 df –h 看到我们自己的设备，它在 dev/nvme0n1 下，被挂载到 /tmp/ene_test/。我们可以进入挂载的文件夹下面对这个设备进行各种磁盘操作了。


（完）


----------


## 参考文献

[1] Li H, Hao M, Tong M H, et al. The case of FEMU: cheap, accurate, scalable and extensible flash emulator[C]//Proc. of 16th USENIX Conference on File and Storage Technologies (FAST). 2018: 83-90.


  [1]: https://www.usenix.org/conference/fast18/presentation/li
  [2]: http://blog.xxiong.me/usr/uploads/2018/12/3841775662.jpg
  [3]: http://blog.xxiong.me/usr/uploads/2018/12/417258747.jpg
  [4]: http://blog.xxiong.me/usr/uploads/2018/12/2558966397.jpg
  [5]: http://blog.xxiong.me/usr/uploads/2018/12/1939032795.jpg
