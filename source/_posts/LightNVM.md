---
title: LightNVM 简介
date: 2018-12-05 11:13:56
tags:
---

LightNVM[1] 是 CNEXLabs 针对 Open Channel SSD（以下简称 OCSSD）在 Linux 内核中的一种实现，分支托管在 [GitHub][1] 上，目前能找到最早的提交记录是 2015-10-29。LightNVM 的程序栈由三层组成，每一层都向用户空间提供了 OCSSD 的抽象（即用户可以直接与这三层进行 IO 交互而不用经过文件系统，后面会细提到）。示意图如下图所示：
!!!
<center>
!!!
![20181130202312.jpg][2]
!!!
</center>
!!!

 - 最上层(3)是对 FTL 的具体实现，其中包含了基本的地址转换、GC 等操作实现，向上呈现为一个逻辑块设备；
 - 中间的 LightNVM 子系统(2)管理整块设备的划分与聚合，向上可以将多个物理设备以一个逻辑设备或一个物理设备划分为多个逻辑设备的形式提供给 FTL；
 - 最后是与驱动程序的对接(3)，我们称为 LightNVM Lower-Level（驱动相关层），用于实现 LightNVM 的命令到具体设备驱动命令的转换。在当前版本（linux-4.12-rc2）中只实现了与 NVMe 驱动的对接。


----------


## LightNVM Lower-Level

启用了 LightNVM 的 NVMe 设备驱动程序使内核模块能够通过 PPA I/O 接口访问 OCSSD。设备驱动程序将设备作为传统的 Linux 块设备呈现到用户空间，允许应用程序通过 ioctls 与设备进行交互。如果 PPA 接口通过 LBA 公开，那么它也可能会相应地发出 I/O。其中，PPA（Physical Page Address）是一种专为 OCSSD 设计的 I/O 接口，它定义了管理级命令将设备的几何信息（channel, die, plane 等）并且让主机端能对 SSD 进行管理，下图是一个是传统逻辑块地址与其进行的对比：
!!!
<center>
!!!
![20181130202313.jpg][3]
!!!
</center>
!!!
由图中可以看出，逻辑地址是一维的，每个地址对应到存储设备的一个 sector；而 PPA 地址则为每个段赋予了不同的涵义，地址最前端是 Channel 的地址，通常 SSD 设备具有数十个到数百个不等的 Channels，PPA 地址的 Channel 段指定了请求要访问的 Channel 地址；以此类推，后面分别是芯片地址 PU、片内的块地址 Block、分层地址 Plane、块内的页地址 Page 以及 Sector。OCSSD 设备接收到 PPA 格式的地址后，无需进行映射，直接根据这个地址就能定位到唯一的 Page 或 Block。PPA 地址中每个字段的宽度不定，通常是在设备加载时，通过获取设备几何结构信息后再计算的，这样能够增加灵活性。


----------


## LightNVM Subsystem

LightNVM 子系统对应到具体的物理设备，为每个设备创建一个实例。其实例在 PPA I/O 接口支持的块设备的基础上初始化。该实例使内核能够通过内部 nvm_dev 数据结构和 sysfs 等来暴露设备的几何结构信息，几何结构信息就是类似上面 PPA 地址中所涉及的信息，例如设备包含多少 channels、channel 中包含多少 PUs、有多少个 blocks 等。这样，FTL 和用户空间的应用程序可以在使用前就了解到设备的底层结构信息。它还使用 blk-mq[2] 设备驱动程序专用的 I/O 接口公开 vector IO 接口，使得应用程序能够通过设备驱动程序有效地下发 vector I/O。


----------


## pblk

物理块设备（pblk）是 LightNVM 中最重要的一个部分，为上层的 target 抽象实现了完全关联的基于主机的 FTL 功能，为上层暴露出了传统块设备的 I/O 接口。其中，每一个 target 就是物理存储设备的逻辑抽象，彼此间独立，由 LightNVM 子系统划分与管理。target 这样的抽象使内核空间模块或用户空间应用程序能够通过高级 I/O 接口（例如由 pblk 提供的 Block I/O 接口的标准接口访问 OCSSD，或由自定义 target 提供的为应用程序定制的接口来进行访问。主要的职责是以下几点：
 - 处理控制器和介质媒体的限制（写缓冲等）；
 - 逻辑地址到物理地址的转换；
 - 错误处理；
 - 垃圾回收；

具体的实现细节已经在论文中体现（[LightNVM: The Linux Open-Channel SSD Subsystem][4]），[官方文档][5]也一直在持续更新。


（完）


----------


## 参考文献

[1] Bjørling M, González J, Bonnet P. LightNVM: The Linux Open-Channel SSD Subsystem[C]//FAST. 2017: 359-374.
[2] Bjørling M, Axboe J, Nellans D, et al. Linux block IO: introducing multi-queue SSD access on multi-core systems[C]//Proceedings of the 6th international systems and storage conference. ACM, 2013: 22.

  [1]: https://github.com/OpenChannelSSD/linux
  [2]: http://blog.xxiong.me/usr/uploads/2018/11/133890102.jpg
  [3]: http://blog.xxiong.me/usr/uploads/2018/11/174123964.jpg
  [4]: https://www.usenix.org/conference/fast17/technical-sessions/presentation/bjorling
  [5]: https://openchannelssd.readthedocs.io/en/latest/``
