---
title: LightNVM 自顶向下的 IO 通路简析
date: 2018-12-05 11:17:10
tags:
---

这篇 IO 通路简析主要偏向于模块之间的衔接，不涉及模块内的细节，主要回答下面的 4 个问题：
1. 在何处向请求队列注册 make request function 函数（这里是 pblk_make_rq）
2. 在何处调用了（1）注册的 make request function 函数
3. 在何处向请求队列注册了 request function 函数（这里是 pci.c 中的 nvme_queue_rq）
4. 在何处调用了（3）注册的 request function 函数


----------


## LightNVM 两个模块及入口的说明

LightNVM 中包含有两个模块——lightnvm subsystem 和 pblk。这两个模块的关系如下图中标红框处：
!!!
<center>
!!!
![20181130202320.jpg][1]
!!!
</center>
!!!

其中 lightnvm subsystem 的代码位于 /drivers/lightnvm/core.c 中，其使用如下方式初始化：
``` C
builtin_misc_device(_nvm_misc);
```

在这个初始化中，定义了如下操作，其中标红的函数用于初始化 target（这里是 pblk 或 rrpc）
``` C
static const struct file_operations _ctl_fops = {
.open           = nonseekable_open,
.unlocked_ioctl = nvm_ctl_ioctl,
.owner          = THIS_MODULE,
.llseek         = noop_llseek,
```

模块入口说明——pblk模块入口函数有以下六个：
``` C
.make_rq    = pblk_make_rq,
.capacity   = pblk_capacity,
.init       = pblk_init,
.exit       = pblk_exit,
.sysfs_init = pblk_sysfs_init,
.sysfs_exit = pblk_sysfs_exit,
```

其中后面五个函数都只是在 nvm_create_tgt 函数或者 __nvm_remove_tgt 中被调用。关于 pblk_make_rq 函数的说明见下一节。


----------


## pblk_make_rq 相关说明（向请求队列注册 pblk_make_rq）

操作系统为每个块设备维护一个 request queue。当一个支持 lightnvm 的 nvme 设备被识别到后，lightnvm 模块将会为 nvme device driver 创建 target，每个 target 由两部分组成——属于一个 target 的内存空间和针对该内存空间的操作（其中定义的操作是同类型 target 共享的）。
这里创建 target 是通过调用第一章节中提到的函数 nvm_ctl_ioctl 实现的。在该函数间接调用的 nvm_create_tgt 中有一步操作如下：
``` C
tqueue = blk_alloc_queue_node(GFP_KERNEL, dev->q->node);
blk_queue_make_request(tqueue, tt->make_rq);
```

通过这两段代码，将 pblk 模块的入口函数 pblk_make_rq（即 make request function）插入到内核为该 pblk 对应的 nvme device 分区维护的请求队列中。


----------


## 当 IO 请求下来时的操作（如何调用 pblk_make_rq）
1. 当一个请求下来后，首先执行 bio_alloc() 用于分配一个新的 bio，之后初始化 bio 描述符。
2. bio 初始化完毕后，内核调用 generic_make_request() 函数，这是通用块层的主要入口点。在该函数中：
  - 首先调用 bdev_get_queue() 获取与请求的块设备相关的请求队列 rq
  - 之后调用 rq->make_request_fn() 将 bio 请求插入请求队列 rq 中
3. 上一步中的 make_request_fn()（即 pblk_make_rq）在 target 初始化的时候已经插入到该请求队列中，因此调用这一步后就进入 lightnvm 模块中。lightnvm 模块与设备驱动相关的代码位于路径 /drivers/nvme/lightnvm.c 中。


----------


## NVMe 向请求队列中注册 request function（nvme_queue_rq）
nvme 为 namespace 初始化 request queue 的操作：
``` C
nvme_alloc_ns() {		//入口函数
    // 其中包含下面这一步操作，用于初始化一个request queue
    ns->queue = blk_mq_init_queue(ctrl->tagset);
```

 在该函数（blk_mq_init_queue）中调用了另一个函数如下：
``` C
blk_mq_init_allocated_queue() {
    // 向request queue中插入nvme的操作函数
    q->mq_ops = set->ops;
```

其中 nvme 模块的 pci.c 文件中定义了操作入口：
``` C
static const struct blk_mq_ops nvme_mq_ops = {
    .queue_rq     = nvme_queue_rq,
    .complete     = nvme_pci_complete_rq,
    .init_hctx    = nvme_init_hctx,
    .init_request = nvme_init_request,
    .map_queues   = nvme_pci_map_queues,
    .timeout      = nvme_timeout,
    .poll         = nvme_poll,
```


----------


## 当 lightnvm 模块处理完成后（如何调用 nvme_queue_rq）

当 lightnvm 模块中与设备驱动相关的逻辑执行完后，lightnvm（nvme/host/lightnvm.c）将会调用 blk_execute_rq_nowait() 函数将请求交由通用块层处理。该函数部分代码如下：
``` C
blk_execute_rq_nowait() {
    if (q->mq_ops) {
        blk_mq_sched_insert_request(rq, at_head, true, false, false);
            return;
```

~~这里有个误导性的地方：不仔细看的话，会以为调用了 __blk_run_queue(q)，而该函数会调用 __blk_run_queue_uncond()~~

继续上面的话题，blk_mq_sched_insert_request() 函数中有如下调用：
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

题外话：其中这里涉及到的请求队列 q 是属于 nvm_dev 结构体中指向的 request queue，其声明在 /include/linux/lightnvm.h 中。nvm_dev 结构体中的 request queue 指针的赋值操作位于文件 /drivers/nvme/lightnvm.c 中的 nvme_nvm_register() 函数中。如下面代码段所示：
``` C
struct request_queue *q = ns->queue;
struct nvm_dev *dev;
dev->q = q;
```

补充：以下是 nvme 模块注册 request_queue 时的操作，可以看到这里的 request_queue 的默认 make request 函数是 blk_mq_make_request() 函数（后面会被 pblk_make_rq 覆盖）
``` C
nvme_alloc_ns()                         // drivers/nvme/host/core.c
    ns->queue = blk_mq_init_queue(ctrl->tagset);
blk_mq_init_queue()                     // block/blk-mq.c
    q = blk_mq_init_allocated_queue(set, uninit_q);
blk_mq_init_allocated_queue()           // block/blk-mq.c
    blk_queue_make_request(q, blk_mq_make_request);
blk_queue_make_request()                // block/blk-settings.c
    q->make_request_fn = mfn;
```


----------


## NVMe 驱动激活 LightNVM 的操作

#### 注册 lightnvm，通过调用 nvme_nvm_register() 实现
``` C
nvme/host/core.c
nvme_alloc_ns()
    if (nvme_nvm_ns_supported(ns, id) && nvme_nvm_register(ns, disk_name, node)) {
        dev_warn(ctrl->dev, "%s: LightNVM init failure\n", __func__);
        goto out_free_id;
```

#### 注销 lightnvm，通过调用 nvme_nvm_unregister() 实现
``` C
nvme/host/core.c
nvme_free_ns()
if (ns->ndev)
    nvme_nvm_unregister(ns);
```

#### 注册 lightnvm_sysfs，通过调用 nvme_nvm_register_sysfs()
``` C
nvme/host/core.c
nvme_alloc_ns()
    if (ns->ndev && nvme_nvm_register_sysfs(ns))
        pr_warn("%s: failed to register lightnvm sysfs group for identification\n", ns->disk->disk_name);
```

#### 注销 lightnvm_sysfs，通过 nvme_nvm_unregister_sysfs()
``` C
nvme/host/core.c
nvme_ns_remove()
    if (ns->ndev)
        nvme_nvm_unregister_sysfs(ns);
```

#### nvme_nvm_ns_supported
``` C
nvme/host/core.c
nvme_alloc_ns()
    if (nvme_nvm_ns_supported(ns, id) && nvme_nvm_register(ns, disk_name, node)) {
        dev_warn(ctrl->dev, "%s: LightNVM init failure\n", __func__);
        goto out_free_id;
```

#### nvme_nvm_ioctl
``` C
nvme/host/core.c
nvme_ioctl()
    #ifdef CONFIG_NVM
        if (ns->ndev)
            return nvme_nvm_ioctl(ns, cmd, arg);
    #endif
```


----------


## 一图流总结
!!!
<center>
!!!
![20181130202330.png][2]
!!!
</center>
!!!
Lightnvm 分别通过用 pblk_make_rq() 替换原始的 generic_make_request() 从而接收 block layer 下发的请求和向下调用 nvme_queue_rq() 从而将请求传递给驱动程序的方式，实现将自身插入到原生的 IO 栈中。

（完）


  [1]: http://blog.xxiong.me/usr/uploads/2018/12/2538886641.jpg
  [2]: http://blog.xxiong.me/usr/uploads/2018/12/4293740320.png
