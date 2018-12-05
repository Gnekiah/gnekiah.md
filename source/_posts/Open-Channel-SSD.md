---
title: Open-Channel SSD 简介
date: 2018-12-05 11:13:17
tags:
---


## SSD 的特点

SSD 设备的存储单元主要是 [NAND Flash][1][1]，按 page 写入，按 block 擦除，一个 block 内有多个 page，并且擦除的次数有限。根据以上的特点，在使用设备时，必然存在这样的情况：在一个 block 中，有部分 page 保存了有效数据，剩下的 page 全被标记为无效，如果要擦除这个 block，就必须首先将其中的有效数据转移到其他 block，然后才能擦除当前 block，这个过程被称为 GC。

在写入数据时，有些区域保存了热数据，对应的 block 会被频繁擦除，达到一定次数后就无法再写入数据，将被标记为坏块；有的区域保存了冷数据，例如备份的资源文件等，这样就会造成磨损的失衡。极端的例子：一块 SSD 其中一半区域被频繁擦除，导致全部被标记为坏块，寿命归零；而另一半的区域由于不常访问，寿命接近满编。如果能够平摊寿命，那么这个 SSD 就能使用更久，平摊寿命这一过程被称为 WL（磨损均衡）。

根据 IO 栈的分层设计思想，同时考虑到设备厂商的各种质保承诺，以及兼容现有的 IO 协议和接口，SSD 被设计成如下图所示的结构：
!!!
<center>
!!!
![20181130162926.jpg][2]
!!!
</center>
!!!
系统驱动与 SSD 控制芯片之间传输的 IO 请求使用的是逻辑地址，需要经过控制芯片中的 FTL（Flash Translation Layer）转换成真实地址，这样就完成了逻辑地址到真实地址的映射。FTL 负责的主要功能有：磨损均衡、地址映射、坏块管理等，可以在一定程度上提高使用寿命，但是也存在负面的影响，例如：

 - 数据迁移、调度、垃圾回收、磨损均衡等都由固件决定, 一旦定制后难以修改，缺少灵活性；
 - 固件只能通过对工作负载的分析进行设计，通用的 SSD 通常只能采取折中的设计方案；
 - 即使大厂可以进行专用 SSD 的定制，工作负载变更将导致定制的 FTL 发挥不出性能；
 - GC、WL、over-provisioning 等操作在一定程度上浪费了 SSD 的存储空间和使用寿命；
 - 由于 FTL 置于“黑匣子”中，无法保证可以预测的请求时延（latency）；


----------


## SSD 的发展趋势

SSD 由于其优异的存储性能，几乎已成为云平台的标配；在个人市场领域，SSD 也同样被广泛地使用。我们知道 SSD 控制器内部算法核心是 FTL，把主机侧逻辑地址转换为 SSD 内部 NAND 芯片的物理地址。一般的消费级 SSD 控制器内置 FTL，因为功能比较简单和统一，对性能的需求也较低，且消费级市场经过 WinTel 联盟多年的锤炼，各种接口非常统一，用户的需求也很单一，只要支持 Intel 主板、Windows 操作系统就可以了，大大简化了各种外设的硬件设计。

但是进入了群雄并立的企业级市场，有各种各样的客户和芯片厂家，大家使用多种多样的操作系统和架构，甚至 Google、Facebook、BAT 等大厂都开始定制硬件、修改内核以支持自身的需求。在这种情况下，FTL 放在 SSD 控制器里面已经难以满足需求，企业用户希望能够根据自己的数据特点设计高效的 FTL，例如：

 - 搜索引擎可以把索引表和 SSD 物理地址对应起来；
 - 日志数据可以直接流式写入 Flash 通道；
 - 数据库希望 key-value 能对应到 SSD 物理地址；


----------


## Open Channel SSD

[Open Channel SSD][3] 是面向企业级市场的其中一个发展方向。在 FAST'2015 闪存峰会上，CNEXLabs 介绍了他们的 Open Channel SSD[2] 概念，通过将 FTL 的功能移动到主机侧的方式，实现一定程度的 FTL 定制，给企业级 SSD 应用提供了新的解决方案。基本架构如下图所示：

!!!
<center>
!!!
![20181130193321.jpg][4]
!!!
</center>
!!!

该方案把 SSD 内部的存储通道开放给主机系统，这样控制器只负责 Flash 数据传输、ECC、错误处理、坏块管理等工作，而 FTL 层的设计由用户自己根据需求实现。这样也更加方便以更高效的方式控制 SSD 阵列，实现了在软件中定义存储。通过 CNEXLabs 公布的实验数据得知，主机侧增加的任务，造成的 latency 只增加了很少的一部分。上图中的 LightNVM[3] 架构是 CNEXLabs 提供的对 Open-Channel 的一种实现。通过把 SSD 中的 FTL 层集成到主机系统中，对工作负载的优化工作就可以在驱动层、块层、文件系统或应用程序本身中进行。


----------


## Open Channel SSD 的优势

根据 CNEXLabs 官方的数据，目前 OCSSD 已经被 WEB 量级的数据中心、超融合基础设施（Hyper-converged Infrastructure）、Flash Array Vendors、高性能计算中心、Tier1 云服务提供商等采用。例如百度使用 OCSSD 来简化一个键值存储的存储堆栈；Fusion-IO 和 Violin 内存均实现主机端存储堆栈来管理 NAND 介质并提供 Block I/O 接口。如下列举一些 OCSSD 的优势：

 - 可预测性
   - 达到一致且可预测的 IO，低时延的 QoS；
   - 减少了写放大问题；
   - 能够实现 IO 隔离，减少了邻间干扰的问题；

 - 方便管理
   - 能够全局地管理所有的 Flash 存储介质；
   - PB 级存储的实现；
   - 单层的地址空间；
   - 根据应用工作负载的特点来配置所期望的 FTL 功能从而更好地优化 IO；


----------


## 总结

OCSSD 与传统的消费级 SSD 在 IO 通路上的区别就在于：

1. 对于传统 SSD 而言，请求从文件系统经过调度最终被驱动程序发出来时，请求中指向的地址是逻辑地址，在 SSD 内部的控制芯片中，被转换成了物理地址，负责这个工作的程序被称为 FTL。
!!!
<center>
!!!
![20181130202317.jpg][5]
!!!
</center>
!!!
2. 对于 OCSSD，请求从文件系统发下来后，首先被转换成了物理地址（或者文件系统发出来就直接就是物理地址），然后再由驱动程序发给 OCSSD 控制器，控制器接收到的请求中的地址是物理地址。因此也被称为 FTL 上移到系统端。

（完）


----------


## 参考文献

[1] Micheloni R, Crippa L, Marelli A. Inside NAND Flash Memories[J]. 2010.
[2] Bjørling M. Open-Channel Solid State Drives[J]. Vault, Mar, 2015, 12: 22.
[3] Bjørling M, González J, Bonnet P. LightNVM: The Linux Open-Channel SSD Subsystem[C]//FAST. 2017: 359-374.


  [1]: https://link.springer.com/content/pdf/10.1007/978-90-481-9431-5.pdf
  [2]: http://blog.xxiong.me/usr/uploads/2018/11/1109110897.jpg
  [3]: https://events.static.linuxfound.org/sites/events/files/slides/LightNVM-Vault2015.pdf
  [4]: http://blog.xxiong.me/usr/uploads/2018/11/3099188414.jpg
  [5]: http://blog.xxiong.me/usr/uploads/2018/12/4056922680.jpg
