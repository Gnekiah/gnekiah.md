---
title: LightNVM 与 Open Channel SSD 的关系以及在 IO 栈上的位置
date: 2018-12-05 11:05:12
tags:
---

Open-Channel SSD 是一种设备，与 SSD 不同之处在于，前者将 SSD 的 FTL(Flash Translation Layer) 提出来，交给主机管理与维护，其优点是：高吞吐，低延迟，高并行。LightNVM 则是 Open-Channel SSD 在主机上的驱动程序扩展。


----------


## OCSSD 的特性

1. I/O 分离：将 SSD 划分为数个 channels, 映射到设备的并行单元上。应用举例：多个应用程序能够同时访问不同的 channels 来实现并行的进行 I/O 操作。
2. 可预测的延迟：通过控制主机何时、向何地址、如何提交 I/O 给 OCSSD 来实现可预测的延迟。
3. 软件定义存储：通过将 SSD 的 FTL 集成到主机中，能够实现根据实际应用的特点，在主机 FTL 中进行负载优化，或者在文件系统中优化，甚至也能够在应用程序中实现。


----------


## LightNVM 在 IO 栈中所处的位置

下图是 LightNVM 的分层结构示意图，直观的来说，OCSSD 即是设备，主机系统通过 NVMe 设备驱动程序与 OCSSD 进行数据的交换。传统的 SSD 通常走 SCSI 驱动，其之上就是 block 层以及文件系统，或者是 NVMe 驱动，直接处理 bio；而 OCSSD 的设备由于已经将 FTL 的功能挪到了主机侧，所以直接接受的请求是物理地址的，因此在驱动程序之上，必须要有一个实现地址转换的程序。图中的 FTL（Block Device Target 和 General Media Manager）就是这个功能的实现。
!!!
<center>
!!!
![20181130202314.jpg][1]
!!!
</center>
!!!
用户空间的应用程序可以通过文件系统访问 OCSSD，在这条通路上，文件系统下发的 bio 将直接被 LightNVM 打包成 nvmrq 的请求格式，这是 LightNVM 定义的请求格式；之后 nvmrq 被下发给 LightNVM 的驱动相关层，驱动相关层实现 nvmrq 到具体驱动程序定义的请求格式的转换。如下图所示，驱动相关层由具体驱动程序加载，当设备支持 OCSSD 时，将自动激活驱动相关层。目前的版本中只支持 NVMe。
!!!
<center>
!!!
![20181130202315.jpg][2]
!!!
</center>
!!!
这样设计的好处是：可以很方便的将 LightNVM 移植到使用其他协议的设备上，例如 UFS 或者 SATA，只需要满足设备为 Open Channel 的设备，然后在驱动中为 LightNVM 的激活和命令转换提供支持即可。


----------


## LightNVM 的源码结构（linux-4.10.3）

- 实现 FTL：rrpc.[ch] (round robin, page-based FTL, and cost-based GC)
- General Media Manager：gennvm.[ch]
- lightnvm core：core.c、include/linux/lightnvm.h
- NVMe 被扩展用于向 LightNVM 设备提供支持：drivers/nvme/host/lightnvm.c 

LightNVM 的核心数据结构如下图所示：
!!!
<center>
!!!
![20181130202316.jpg][3]
!!!
</center>
!!!
每块物理的 OCSSD 都会通过 NVMe 驱动程序激活设备相关层，并在 General Media Manager 中为其创建一个 nvm_dev 的结构体实例和 gen_dev 实例，代表一块完整的设备；nvm_dev 可以被划分为多个 target，类似于磁盘分区，不同之处在于，LightNVM 只能按照 luns 为基本单位进行划分。每个 target 对应到一个 nvm_tgt_dev 的结构体，以及用于处理这个分区的负责实现 FTL 功能的 target type，目前实现的 type 只有 rrpc 一种。

（完）

  [1]: http://blog.xxiong.me/usr/uploads/2018/12/3566424873.jpg
  [2]: http://blog.xxiong.me/usr/uploads/2018/12/816566981.jpg
  [3]: http://blog.xxiong.me/usr/uploads/2018/12/2329710792.jpg
