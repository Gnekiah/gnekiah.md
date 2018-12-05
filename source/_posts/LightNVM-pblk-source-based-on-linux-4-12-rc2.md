---
title: LightNVM - pblk 源码解析（基于 linux-4.12-rc2）
date: 2018-12-05 11:15:03
tags:
---


## 基本结构图

下图是 pblk 的结构图，其中给出了 read 请求和 write 请求的操作示意图，以及 pblk 核心的几个数据结构——L2P table、write buffer、write context。关于具体操作的细节我们在后面的小节给出：
!!!
<center>
!!!
![20181130202318.png][1]
!!!
</center>
!!!

1. Buffer 包含两块内容：write buffer 和 write context。write buffer 存储 4KB 为单位(sector)的 data entries，其中write buffer 的大小为 page 的大小 \* PUs 的数量 \* 8；write context 存储每个 entry 的 metadata。
2. L2P table 是 target 的逻辑地址到物理地址的映射，这是 pblk 中的映射，在请求下发给 nvm manager 后 target 的物理地址会再被映射为 device 的物理地址。
3. write thread 是一个单独的线程，在 lightnvm 中，write buffer 有多个“Producers”和 一个“Consumer”，其中的 Consumer 就是这里的 write thread。write thread 在两种情况下会被唤醒：
  - 定时器触发；
  - write buffer 达到 flush 的条件(主动唤醒);


----------


## line 的概念

line：从 target 的每个 lun 中取一个 block，组成一个 line。下图是一个例子，帮助我们直观的认识 line。在这个 target 中有 4 个 luns，每个 lun 中有 6 个 blocks，在这个例子中，一共有 6 个 lines，每个 line 的大小是 4 个 blocks：
!!!
<center>
!!!
![20181130202319.png][2]
!!!
</center>
!!!
 - line 0 包含的 blocks 编号为：0，6，12，18
 - line 1 包含的 blocks 编号为：1，7，13，19
 - ....................................


----------


## discard 请求

discard 的作用是使请求的数据无效化。discard 是针对 L2P table 的操作，只需要将 L2P table 的逻辑地址对应的物理地址设为 empty 就实现了将目标数据无效化的操作。涉及到 discard 的操作如下：
!!!
<center>
!!!
![20181130202320.png][3]
!!!
</center>
!!!
当 discard 请求发送到来时，首先查找 L2P table，找到对应的物理地址；之后查找 write buffer，如果 cache hit，则直接更新 L2P table 将请求的逻辑地址对应的物理地址标记为无效；如果 cache miss，则获取请求页地址所在的 line，将其标记为不可用（确保在该地址被无效化之前不会被访问），之后更新 L2P table。相关 discard 的操作流程见下方的流程图：
!!!
<center>
!!!
![20181130202321.png][4]
!!!
</center>
!!!


----------


## read 请求

read 操作大致可分为三步：
1. 从 L2P table 查找物理地址；
2. cache hit，则从 buffer 中读取；
3. cache miss，则从设备读取；

注意：如果只有部分请求 cache hit，则要将 cache miss 的请求重新构造成新的请求，再从设备读取。在这种情况下，当请求从设备中读取数据后，要将 new  bio 中的数据拷贝到 original bio 中。（所有请求都 cache miss 的情况下，从设备中读取数据后不执行该步骤。）
!!!
<center>
!!!
![20181130202322.png][5]
!!!
</center>
!!!

下面的流程图展示了整个 read 操作的过程。对于到达的 read 请求，首先查找 L2P table，获取该请求的物理地址；然后查找 write buffer，当 cache hit 的情况下，将 write buffer 中的数据拷贝到 bio 中。这里牵涉到两种情况：只有部分请求页 cache hit 和 所有请求页都 cache hit。如果是所有请求页都 cache hit，则 end io；如果只有部分请求页 cache hit，那么需要将 cache miss 的请求页重新狗造成一个新的请求，然后下发给设备用于读取数据。
!!!
<center>
!!!
![20181130202323.png][6]
!!!
</center><center>
!!!
![20181130202324.png][7]
!!!
</center>
!!!
在向设备发送请求时，如果是请求页全部 cache miss 的情况，则直接提交请求；如果是部分 cache miss 的情况，则需要先将 cache miss 的请求页拼接成一个新的请求，然后下发给设备，直到设备读取数据并返回后，需要将上一步构造的新的 bio 中的数据拷贝到原始请求的 bio 中。
!!!
<center>
!!!
![20181130202318.jpg][8]
!!!
</center>
!!!


----------


## write 请求

write 操作分为两个部分：
1. 写入 write buffer；
2. write thread从 write buffer 写入设备；

在整个写入的操作中，对buffer的操作可以分为两类角色：多个生产者和一个消费者。所有的写操作都会将数据写入 write buffer（扮演生产者），然后由write thread将数据写入设备中（扮演消费者）。下面是这两个部分的具体操作流程：
!!!
<center>
!!!
![20181130202325.png][9]
!!!
</center>
!!!

写入 buffer 的主要操作如下：
1. 判断能否向 buffer 写入请求的 entries；
2. 向 buffer 中写入 entries；
3. 更新 L2P table；
4. 判断是否需要唤醒 write thread；

下面的流程图展示了向buffer中写数据的操作过程。当write请求到来时，首先要判断有没有write buffer足够空间用来进行本次的写请求，如果空间不够，需要进行io调度，将buffer中的数据写回设备中。如果空间足够，就将数据写入到write buffer中，并更新write context，其中write buffer中的单位是entry，大小是4KB。最后再更新 L2P table。这里在end io之前需要判断是否需要唤醒write thread将buffer中的数据写入到设备。
!!!
<center>
!!!
![20181130202326.png][10]
!!!
</center>
!!!

从 buffer 写入设备的主要操作：
1. 计算要写回的 entries 数量；
2. 将 entries 添加到 bio 构造 write request；
3. nvme submit io向设备提交请求；

下图是write thread进行的操作流程：
!!!
<center>
!!!
![20181130202327.png][11]
!!!
</center>
!!!

write thread 被唤醒的条件有两个：
1. 被定时器触发；
2. write buffer 中需要写回到设备的数据量达到了阈值；

当 write thread 被唤醒后，首先计算 write buffer 中需要同步的 entries 的总数，这些是需要写回设备的数据单元。之后将 entries 中的数据添加到 bio，用于向设备发送写请求。这里需要注意，write thread 不需要更新 L2P table，因为这个操作在前半部分的 write to buffer 中已经完成了。


----------


## GC

GC 能够充分将存储单元利用起来。GC thread 被唤醒的条件有：
1. 定时器唤醒；
2. write thread 主动唤醒 GC thread；
!!!
<center>
!!!
![20181130202328.png][12]
!!!
</center>
!!!

在GC thread被唤醒后，只遍历每个line，并初始化每个line 的工作队列。如上图所示，gc full list 中保存的是 lines，只有全部 full 的 line 才会加入 gc full list。停止 gc 的条件是 free blocks 数量达到预设的阈值。GC是由line管理的，内核遍历由line中的工作队列组成的工作队列链表，对每一个line，分别执行GC 操作。在这个版本的lightnvm中，GC的操作是简单的将数据读出来保存到write buffer中。关于擦除块的操作在write thread中。
!!!
<center>
!!!
![20181130202329.png][13]
!!!
</center>
!!!


----------


## L2P table recovery

L2P table 是 lightnvm 一个非常核心的数据结构，这张表建立起从逻辑地址到存储器物理地址的映射关系，因此该表是保存在设备的 oob 部分的，当设备初始化的时候直接向设备发送读取 l2p 的请求，即可读取 L2P 表。

（完）


  [1]: http://blog.xxiong.me/usr/uploads/2018/11/2440312904.png
  [2]: http://blog.xxiong.me/usr/uploads/2018/11/3321758576.png
  [3]: http://blog.xxiong.me/usr/uploads/2018/11/1260758134.png
  [4]: http://blog.xxiong.me/usr/uploads/2018/11/3492987467.png
  [5]: http://blog.xxiong.me/usr/uploads/2018/11/1384739315.png
  [6]: http://blog.xxiong.me/usr/uploads/2018/11/2653169438.png
  [7]: http://blog.xxiong.me/usr/uploads/2018/11/3328136321.png
  [8]: http://blog.xxiong.me/usr/uploads/2018/12/2697667031.jpg
  [9]: http://blog.xxiong.me/usr/uploads/2018/11/2557055257.png
  [10]: http://blog.xxiong.me/usr/uploads/2018/11/1387548915.png
  [11]: http://blog.xxiong.me/usr/uploads/2018/11/2770234034.png
  [12]: http://blog.xxiong.me/usr/uploads/2018/11/3960561637.png
  [13]: http://blog.xxiong.me/usr/uploads/2018/11/649340819.png
