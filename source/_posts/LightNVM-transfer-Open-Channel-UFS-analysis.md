---
title: LightNVM 移植到 Open Channel UFS 设备的实现分析
date: 2018-12-05 11:16:02
tags:
---


基于 UFS 的 Open Channel FTL 实现与基于 NVMe 的实现思路类似，可按层划分为三个大步骤，自下而上分别为：

1. UFS 设备侧的 FTL 相关功能修改；
2. 主机侧 UFS 驱动程序的操作命令扩展；
3. 主机侧软件定义的 FTL 功能实现。

此外我们还需要一个工具用于获取运行数据和验证，例如获取设备的详细参数，例如 channels,luns,blocks,pages 等固有属性，以及 bad block table 和 mapping table 等运行数据。


----------


## 模拟 UFS 设备

UFS 设备侧的修改包括：移除 Garbage Collection、Wear-leveling、Translation Map、Bad Block Management 等需要上移至 Host 端的功能，保留 ECC 等不适合上移的功能，并添加以下几个必要的操作：
- get device geometry
- get l2p table
- set l2p table
- get bad block table
- set bad block table
- erase block

但是由于目前没有可用的实体 Open Channel UFS 设备，我们只能够寻求通过在虚拟机上模拟一个类似设备的方式来实现原型。在衡量众多虚拟机程序后，我们将目标锁定在 Qemu。主要原因有两点：
1. Lightnvm 的研究团队用于实验的环境中使用了 Qemu，并且该团队在 Qemu 提供的 NVMe 仿真程序的基础上进行了扩展，实现了对 Open Channel SSD 的支持，并且该扩展的项目 Qemu-nvme 已开源；
2. 常年保持活跃的 Qemu 社区和规整的代码更有利于开发。

由于 Qemu 目前尚未支持 UFS 设备的模拟，因此我们还需要查阅 JEDEC 提供的 UFS STANDARD。我们的目标是 Open Channel UFS，因此我们只需要实现我们需要的上面 6 个命令，外加 Read,Write 命令，一共 8 个命令的操作即可。


----------


## UFS 驱动扩展

主机侧 UFS 驱动程序保持原有的大框架不变，在 UFS Application（UAP）这层的 Command Set 中，新添加 6 条命令用以扩展驱动程序，这个部分扩展的命令与设备侧新添加的功能一一对应，即：
- get device geometry
- get l2p table
- set l2p table
- get bad block table
- set bad block table
- erase block

除了扩展 6 个新的命令，由于设备接收自主机的地址不再是逻辑地址，因此还需要重新定义主机侧与设备之间适用于物理地址的地址接口格式。首先，从主机发出的地址不再是逻辑上的线性地址，取而代之的是由 channel,lun,block,page 等拼成的新的组织形式，因此存在地址可能无效的情况，也可能目标地址位于坏块中，这里我们假定所有处理都由 Host 端的 FTL 处理。


----------


## Lightnvm 子系统移植
主机侧软件定义的 FTL 实现可参考 Linux-4.12 中已存在的 Lightnvm 子系统。我们可对其做少许修改使之能够与 UFS 驱动程序通信。最后实现的预期效果如下，对 Device 与 Driver 做部分修改，并在 Driver 以上，Block Layer 以下插入一个Lightnvm 子层：
!!!
<center>
!!!
![20181130202319.jpg][1]
!!!
</center>
!!!

- 从Block Layer传下来的请求仍然是逻辑地址；
- 在Lightnvm中将该逻辑地址映射成物理地址；
- Driver照常提交包含物理地址的请求；
- Device接收到请求后直接对该请求的地址进行操作。

这里新扩展的6条命令的作用在于：
- Lightnvm需要知道设备的属性，因此需要get device geometry
- 设备初始化时需要对设备中已有的数据构建映射关系，需要get l2p table
- 设备卸载时需要保存映射关系，需要set l2p table
- 设备中某些块可能是坏块，无法写入数据，需要get bad block table
- 设备使用过程中会产生坏块，需要set bad block table
- 当Lightnvm做GC时，有些块要擦除，需要erase block

最后，验证原型需要单独开发一套工具，比如获取设备属性、运行过程中设备状态和各种 table 的数据等，这需要我们向 uapi 中添加相应的函数。

（完）

  [1]: http://blog.xxiong.me/usr/uploads/2018/12/4159062797.jpg
