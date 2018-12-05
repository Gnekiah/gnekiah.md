---
title: LightNVM 代码分析总结
date: 2018-12-05 11:23:32
tags:
---

内核版本：Linux-4.19


----------


## Lightnvm 与 pblk

Lightnvm 与 pblk 的关系，类似于 linux 块层与 IO 调度器之间的关系。即在 lightnvm 中可以有多种 FTL 的实现，这里 pblk 就是一种 FTL 的实现。Lightnvm 子系统在支持PPA接口的块设备的基础上进行初始化。该模块使内核能够通过内部 nvm_dev 数据结构和 sysfs 等来暴露设备的几何结构信息。通过这种方式，FTL 和用户空间的应用程序可以在使用前就了解到设备的底层信息。此外 Lightnvm 子系统还有一个最重要的功能——管理 target 的划分以及指定用于管理 target 的 FTL。


----------


## Lightnvm 的三层结构

如下图所示，Lightnvm 被分为三个部分 ③②①。自上而下分别为 FTL、Lightnvm 子系统以及设备驱动程序。Lightnvm 子系统通过将 FTL 提供出来的 make_rq 函数指针替换到块层的 blk_queue_make_request，从而实现直接处理 bio 而不需要经过 IO 调度器。

当一个 bio 从文件系统发出来后，首先进入 FTL 模块，对应下图中的 ③；在 FTL 中处理完毕后，如果要与设备进行交互，则 FTL 必须要将请求下发到设备。由于 FTL 不知道底层设备类型，故也无法确定设备驱动程序所定义的数据格式，同时也为了确保 Lightnvm 能够对多种不同类型的设备都兼容，因此 Lightnvm 与具体设备驱动之间采取了一个中间件，叫做 Lightnvm 驱动相关层，其功能是将 Lightnvm 所定义的命令格式转换成具体设备驱动程序的命令格式，再提交给对应的驱动程序。由此可知，如果某个设备要使用 Lightnvm 模块，则该设备对应的驱动程序必须要实现上面提到的中间件。
!!!
<center>
!!!
![20181130202355.jpg][1]
!!!
</center>
!!!

Lightnvm 与具体设备驱动的中间件如下图示，Lightnvm 有一套通用的命令格式和请求格式，中间层只起到格式转换的作用。
!!!
<center>
!!!
![20181130202356.jpg][2]
!!!
</center>
!!!

使用 Lightnvm 的设备驱动程序使内核其他模块能够通过 PPA I/O 接口直接访问设备。设备驱动程序将设备作为传统的 Linux 块设备公开到用户空间，允许应用程序通过 ioctls 与设备进行交互。PPA 地址格式定义见下图：
!!!
<center>
!!!
![20181130202357.jpg][3]
!!!
</center>
!!!


----------


## pblk (FTL 实现)的源码分析

pblk 是对 Lightnvm 中负责具体 FTL 功能的一种实现，在旧版本中还有 rrpc，但是在最新的内核 4.19 版本中，已经移除了 rrpc 的模块，现在只有 pblk 这一种实现。从管理范围的范畴来说，pblk 是负责管理 target 的，一个设备可以切分成多个 target，而每个 target 可以使用不同的 FTL 的具体实现来进行管理。抽象出 target 这一方式，使内核空间模块或用户空间应用程序能够通过高级 I/O 接口（例如由 pblk 提供的块 I/O 接口的标准接口）访问设备，或由自定义 target 提供的为应用程序定制的接口来进行访问。其中，每一个 target 就是一块物理存储设备的抽象，每个 target 可以单独使用一种类型的 FTL 来管理，彼此间独立。pblk 主要的职责是以下几点：
 - 提供主机侧的写缓冲机制
 - 实现从主机逻辑地址到设备物理地址的转换
 - 处理本该由设备进行的垃圾回收


----------


## 模块之间的衔接

从源码的角度分析，将 Lightnvm 插入原生的驱动程序与块层之间，需要解决两个交互点——块层与 Lightnvm 之间的衔接以及 Lightnvm 与驱动程序之间的衔接。这里我们将两个交互点整理成 2 个问题：
1. 在何处向请求队列注册 make request function 函数
2. 在何处向请求队列注册 request function 函数

其中注册 make request 函数是为了将能够直接处理文件系统中的 bio；注册 request 函数则是为了能够将请求发往具体驱动程序。下面分别对每个问题进行源码分析。

#### 向请求队列注册 pblk_make_rq

操作系统为每个块设备维护一个 request queue。当一个支持 Lightnvm 的 nvme 设备被识别到后，lightnvm 模块将会为 nvme device driver 创建 target（类比于磁盘的分区），一个 Lightnvm 设备可以划分多个 target（要求划分的 target 的起始 channel 地址对齐，channel 结束地址没有要求），划分 target 的基本单位为 lun，并且不同的 target 可以采用不同的 FTL 的实现（例如 target 1 使用 pblk 管理，target 2 使用 rrpc 管理）。

这里创建 target 是通过调用函数 nvm_ctl_ioctl 实现的。在该函数中又间接调用了 nvm_create_tgt，其中有一步重要操作如下：
``` C
tqueue = blk_alloc_queue_node(GFP_KERNEL, dev->q->node);
blk_queue_make_request(tqueue, tt->make_rq);
```

通过这两段代码，将 pblk 模块的入口函数 pblk_make_rq（即 make request function）插入到内核为该 pblk 对应的 nvme device 分区维护的请求队列中。

#### 当 IO 请求下来时，如何调用 pblk_make_rq

当 IO 请求从 block layer 下来后，执行下面的步骤将请求传给 Lightnvm 层。
1. 当一个请求下来后，首先执行 bio_alloc() 分配一个新的 bio 并初始化 bio 描述符；
2. bio 初始化完毕后，内核调用 generic_make_request() 函数；
3. 在该函数中，调用 bdev_get_queue() 获取与请求的块设备相关的请求队列 rq；
4. 之后调用 rq->make_request_fn() 将 bio 请求插入请求队列 rq 中；

第 4 个步骤的 make_request_fn()（即 pblk_make_rq）在 target 初始化的时候已经插入到该请求队列中，因此调用这一步后就进入 lightnvm 模块中。lightnvm 模块与设备驱动相关的代码位于路径 /drivers/nvme/host/lightnvm.c 中。

#### NVMe 向请求队列中注册 nvme_queue_rq

nvme 为 namespace 初始化 request queue 的操作：

 - 入口函数，其中包含下面这一步操作，用于初始化一个 request queue
``` C
nvme_alloc_ns() {
    ns->queue = blk_mq_init_queue(ctrl->tagset);
```

 - 在该函数 (blk_mq_init_queue) 中调用了另一个函数如下，向 request queue 中插入 nvme 的操作函数：
``` C
blk_mq_init_allocated_queue() {
    q->mq_ops = set->ops;
```

 - 其中 nvme 模块的 pci.c 文件中定义了操作入口：
``` C
static const struct blk_mq_ops nvme_mq_ops = {
    .queue_rq     = nvme_queue_rq,
    .init_request = nvme_init_request,
    .......
```

#### 当 lightnvm 处理完后，如何调用 nvme_queue_rq

当 Lightnvm 中与驱动相关的逻辑执行完成后（从 lightnvm request 生成特定驱动的命令），Lightnvm 的驱动相关层（nvme 中位于 drivers/nvme/host/lightnvm.c）将会调用 blk_execute_rq_nowait() 函数将请求交由通用块层处理。该函数部分代码如下：
``` C
blk_execute_rq_nowait() {
    if (q->mq_ops) {
        blk_mq_sched_insert_request(rq, at_head, true, false, false);
        return;
```

在 blk_mq_sched_insert_request() 函数中有如下调用关系：
``` C
blk_mq_sched_insert_request() {
    blk_mq_run_hw_queue(hctx, async);
blk_mq_run_hw_queue() {
    __blk_mq_delay_run_hw_queue(hctx, async, 0);
__blk_mq_delay_run_hw_queue() {
    __blk_mq_run_hw_queue(hctx);
__blk_mq_run_hw_queue() {
    blk_mq_sched_dispatch_requests(hctx);
blk_mq_sched_dispatch_requests() {
    blk_mq_dispatch_rq_list(q, &rq_list);
blk_mq_dispatch_rq_list() {
    // 这里调用了nvme模块中的nvme_queue_rq
    ret = q->mq_ops->queue_rq(hctx, &bd);
```


----------


## read

pblk 的读写过程见下图示，从整体的角度看，读请求和写请求都要首先对 L2P 表进行操作，之后再操作 write buffer。下面分别对 read 和 write 的过程进行详解。
!!!
<center>
!!!
![20181130202358.jpg][4]
!!!
</center>
!!!
read 操作大致可分为三步：
1. 从 L2P table 查找物理地址；
2. 从 buffer 中查找该地址，若查找 cache 命中，则从 buffer 中读取；
3. 如果查找 buffer 未命中，则构造读请求并从设备读取数据。

如果只有部分请求命中，则要将未命中的请求重新构造成新的请求，再从设备读取。在部分命中的这种情况下，当从设备中读取数据后，需要将 buffer 中的命中的那部分数据与从设备中读取的未命中的数据进行合并，再返回给文件系统。
!!!
<center>
!!!
![20181130202359.jpg][5]
!!!
</center>
!!!


----------


## write

write 操作分为两个部分：
1. 将文件系统下发的数据写入到 buffer；
2. write thread 将数据从 buffer 写入到设备。

在整个写入的操作中，对 buffer 的操作可以分为两类角色：多个生产者和一个消费者。
 - 所有的写操作都会先将数据写入到 buffer 中；
 - 然后由 write thread 将数据写入设备中。下面是这两个部分的具体操作流程。

1. 写入 buffer 的主要操作如下：
 - 判断能否向 buffer 写入请求数据；
 - 将数据写入到 buffer 中；
 - 写入数据后，更新 L2P table；
 - 判断是否需要唤醒 write thread。
2. 从 buffer 写入设备的主要操作：
 - 计算要写回的 entries 数量；
 - 将 entries 添加到 bio 并构造 request；
 - 向设备提交请求。

下面的流程图展示了向 buffer 中写数据的操作过程。当 write 请求到来时，首先要判断有没有 buffer 足够空间用来进行本次的写请求，如果空间不够，需要触发 write thread，将 buffer 中的数据写回设备中。如果空间足够，就将数据写入到 buffer 中，其中 buffer 中的单位是 entry，大小是 4KB。最后再更新 L2P table。这里在 end io 之前需要判断是否需要唤醒 write thread 将 buffer 中的数据写入到设备。
!!!
<center>
!!!
![20181130202360.jpg][6]
!!!
</center>
!!!

write thread 被唤醒的条件有两个：
1. 被定时器触发；
2. buffer 中需要写回到设备的数据量达到了阈值。

当 write thread 被唤醒后，首先计算 buffer 中需要同步的 entries 的总数，这些是需要写回设备的数据单元。之后将 entries 中的数据添加到 bio，用于向设备发送写请求。这里需要注意，write thread 不需要更新 L2P table，因为这个操作在前半部分的 write to buffer 中已经完成了。


----------


## discard

discard 的作用是使请求的数据无效化。discard 是针对 L2P table 的操作，只需要将 L2P table 的逻辑地址对应的物理地址设为 empty 就实现了将目标数据无效化的操作。涉及到 discard 的操作如下： 
1. 当 discard 请求发送到来时：
2. 首先查找 L2P table，找到对应的物理地址；
3. 之后查找 write buffer；
4. 如果 cache hit，则直接更新 L2P，将请求的逻辑地址对应的物理地址标记为无效；
5. 如果 cache miss，则获取请求页地址所在的 line，将其标记为不可用（确保在该地址被无效化之前不会被访问），之后更新 L2P table。
!!!
<center>
!!!
![20181130202361.jpg][7]
!!!
</center>
!!!


----------


## GC

GC 能够充分将存储单元利用起来。GC thread 被唤醒的条件有：
1. 定时器唤醒；
2. write thread 主动唤醒 GC thread。

在 GC thread 被唤醒后，只遍历每个 line，并初始化每个 line 的工作队列。如上图所示，gc full list 中保存的是 lines，只有全部 full 的 line 才会加入 gc full list。停止 gc 的条件是 free blocks 数量达到预设的阈值。GC 是由 line 管理的，内核遍历由 line 中的工作队列组成的工作队列链表，对每一个 line，分别执行 GC 操作。在当前版本的 lightnvm 中，GC 的操作是简单的将数据读出来保存到 write buffer 中。关于擦除块的操作在 write thread 中。
!!!
<center>
!!!
![20181130202362.jpg][8]
!!!
</center>
!!!


----------


## Buffer 管理

Buffer 的相关数据结构如下：
``` C
struct pblk_rb_entry {
    struct ppa_addr cacheline;  // entry相对于buffer的地址
    void *data;                 // 指向数据
    struct pblk_w_ctx w_ctx;    // entry的上下文

struct pblk_w_ctx {
    struct bio_list bios;       // 用于回调
    u64 lba;                    // 相对于文件系统的逻辑地址
    struct ppa_addr ppa;        // 相对于设备的物理地址
    int flags;
```

buffer 以 2 的整数次方的大小进行循环管理，buffer 设定的最小值为二进制 110100100，即 420，且向上取 2 的 n 次方整，即 buffer 能取到的最小的大小为 512。实际取值为成功读取所需要的最小距离乘以 * luns 的数量，向上取 2 的 n 次方整。


----------


## Wear-Leveling

版本 4.19 中没有见到有关于 wear-leveling 的代码。


----------


## Lightnvm 驱动相关层

相较于内核 4.12，当前的 4.19 在 Lightnvm 驱动相关层的源码没有太大变化，源码位于驱动源码文件所在的目录： /drivers/nvme/host/lightnvm.c。在驱动相关层源码中，需要实现下面的操作：
``` C
static struct nvm_dev_ops nvme_nvm_dev_ops = {
    .identity          = nvme_nvm_identity,
    .get_l2p_tbl       = nvme_nvm_get_l2p_tbl,
    .get_bb_tbl        = nvme_nvm_get_bb_tbl,
    .set_bb_tbl        = nvme_nvm_set_bb_tbl,
    .submit_io         = nvme_nvm_submit_io,
    .create_dma_pool   = nvme_nvm_create_dma_pool,
    .destroy_dma_pool  = nvme_nvm_destroy_dma_pool,
    .dev_dma_alloc     = nvme_nvm_dev_dma_alloc,
    .dev_dma_free      = nvme_nvm_dev_dma_free,
    .max_phys_sect     = 64,
```

前四条操作分别对应 admin IO 的四条命令，即：
1. 获取设备的 geometry 信息；
2. 获取 L2P 表；
3. 获取坏块表；
4. 更新坏块表。

第五条 submit_io 用于响应 Lightnvm 传来的常规 IO 请求。后面四条是分配和回收内存相关的操作。最后一条 max_phys_sect 是设备支持的最大物理扇区数。


----------


## IO 命令

#### identity
首先将 Lightnvm request 转换成 NVMe command，然后直接调用 nvme 模块提供的 nvme_submit_sync_cmd 函数（在这其中初始化了一个新的 request），然后执行了这个 request。
#### get_l2p_tbl
以 lun 为单位，同 identity 一样，首先要转换成 NVMe command，然后调用 nvme 提供的 nvme_submit_sync_cmd 函数去处理这个请求。如果请求包含了多个 luns，那么对每个 lun 都需要构造 NVMe command 并提交。
#### get_bb_tbl
同identity。
#### set_bb_tbl
同identity。
#### read/write
General IO（除了 Admin 以外的 IO）的处理方式与 Admin IO 的处理方式大同小异，首先要构造 NVMe command，然后根据 command 和 bio 初始化生成 request，最后调用块设备层的 blk_execute_rq_nowait() 函数执行请求。


----------


## Driver 激活 Lightnvm 驱动相关层

1. 注册 lightnvm，在 nvme 初始化时通过调用 nvme_nvm_register() 实现。（/drivers/nvme/host/core.c）
``` C
nvme_alloc_ns()
    if (nvme_nvm_ns_supported(ns, id) && nvme_nvm_register(ns, disk_name, node)) {
        dev_warn(ctrl->dev, "%s: LightNVM init failure\n", __func__);
        goto out_free_id;
```
2. 注销 lightnvm，在 nvme 驱动被卸载前通过调用 nvme_nvm_unregister() 实现。（/drivers/nvme/host/core.c）
``` C
nvme_free_ns()
if (ns->ndev)
    nvme_nvm_unregister(ns);
```
3. 注册 lightnvm_sysfs，在 nvme 初始化时通过调用 nvme_nvm_register_sysfs()。（/drivers/nvme/host/core.c）
``` C
nvme_alloc_ns()
    if (ns->ndev && nvme_nvm_register_sysfs(ns))
        pr_warn("%s: failed to register\n", ns->disk->disk_name);
```
4. 注销 lightnvm_sysfs，在 nvme 驱动被卸载前调用 nvme_nvm_unregister_sysfs()。（/drivers/nvme/host/core.c）
``` C
nvme_ns_remove()
    if (ns->ndev)
        nvme_nvm_unregister_sysfs(ns);
```

（完）


  [1]: http://blog.xxiong.me/usr/uploads/2018/12/1939970917.jpg
  [2]: http://blog.xxiong.me/usr/uploads/2018/12/4174733462.jpg
  [3]: http://blog.xxiong.me/usr/uploads/2018/12/3526174751.jpg
  [4]: http://blog.xxiong.me/usr/uploads/2018/12/3945780743.jpg
  [5]: http://blog.xxiong.me/usr/uploads/2018/12/896036978.jpg
  [6]: http://blog.xxiong.me/usr/uploads/2018/12/1745448200.jpg
  [7]: http://blog.xxiong.me/usr/uploads/2018/12/1279183542.jpg
  [8]: http://blog.xxiong.me/usr/uploads/2018/12/3129676206.jpg
