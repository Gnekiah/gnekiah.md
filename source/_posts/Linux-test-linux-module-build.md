---
title: Linux 实验内容设计：向内核中插入虚拟块设备
date: 2018-12-05 11:19:04
tags:
---


大三【Linux 操作系统】课的实验内容设计二：向内核中插入虚拟块设备。

事情是这样的，课题教学组有位老师抱怨实验无趣，4 个实验课内容从古沿用至今，每学期的助教都是直接拿往届的实验内容去上，都尽量能少一事就少一事，然后再加上已经上了的实验一，是一如既往的“编译内核与添加系统调用”，被班里的大神们抱怨没啥意思。作为萌新的我只能龟缩在角落玩扫雷，并不断的观察着黑下脸的任课老师。大概这位老师觉得被学生看扁了很不爽吧，次日五个教学班的助教都被召集，要求在一周内设计出有意思的实验内容，并要求附上实验操作手册。~~于是就有了这篇麻烦。~~权当做个记录。


----------


## 实验目的
- 了解 Linux 模块化编程的方式和原理。
- 了解并掌握如何向 Linux 内核中注册一个功能模块。
- 了解 sysfs 文件系统的原理和作用。
- 学习并掌握如何通过内核提供的 API 向 sysfs 中注册自定义节点。
- 了解 Makefile 的编写规则。
- 了解并掌握模块化编译的原理和步骤。
- 学习 insmod 和 rmmod 插入/卸载模块的原理的操作过程。
- 了解 Linux 设备驱动模型。
- 理解设备驱动模型中总线、设备、驱动三者的关系。
- 了解设备的注册机制和原理。
- 掌握块设备的注册原理和操作过程。
- 学习并掌握如何通过内核提供的API向内核中注册块设备。
- 掌握dmesg的使用。


----------


## Linux 内核模块机制

#### 模块结构体——module

module 结构体的主要成员如下代码段所示，module_state 指示模块的状态；模块通过 list_head 将自身插入到内核模块链表中；name 是模块的名称，内核通过这个 name 来识别模块；version 和 srcversion 是模块版本号；之后是模块的入口函数和注销函数——init 和 exit。在实际使用 Linux 内核模块编程时，都必须要实现 init 和 exit 函数。
``` C
struct module {
    ......
    enum module_state state;
    struct list_head list;	
    char name[MODULE_NAME_LEN];
    const char *version;
    const char *srcversion;
    int (*init)(void);
    void (*exit)(void);
    ......
};

enum module_state {
    MODULE_STATE_LIVE,      /* Normal state. */
    MODULE_STATE_CONMING,   /* Full formed, running module_init. */
    MODULE_STATE_GOING,     /* Going away. */
    MODULE_STATE_UNFORMED,  /* Still setting it up. */
};
```

#### 查看模块信息

使用 readelf 命令可以查看模块文件的信息。例如下代码段所示：
``` bash
root@AE86_<darkelf>(lab1)readelf -h darkelf.ko
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             	ELF64
  Data:                              	2's complement, little endian
  Version:                           	1 (current)
  OS/ABI:                            	UNIX - System V
  ABI Version:                       	0
  Type:                              	REL (Relocatable file)
  Machine:                           	Advanced Micro Devices X86-64
  Version:                           	0x1
  Entry point address:               	0x0
  Start of program headers:          	0 (bytes into file)
  Start of section headers:          	332976 (bytes into file)
  Flags:                             	0x0
  Size of this header:               	64 (bytes)
  Size of program headers:           	0 (bytes)
  Number of program headers:        	0
  Size of section headers:			64 (bytes)
  Number of section headers:   		39
  Section header string table index:	38
```

#### 模块加载

模块加载是通过系统调用 init_module 来实现的，该系统调用源码位置位于源码文件 linux/kernel/module.c 中。这个系统调用是通过 SYSCALL_DEFINE3 (init_module...) 实现，其中有两个关键的函数调用。load_module 用于模块加载，do_init_module 用于回调模块的 init 函数。
``` C
SYSCALL_DEFINE3(init_module, void __user *, umod,
        unsigned long, len, const char __user *, uargs)
{
    ......
    return load_module(&info, uargs, 0);
}
	
static int load_module(struct load_info *info, 
        const char __user *uargs, int flag)
{
    ......
    err = module_sig_check(info, flags);
    err = elf_header_check(info);
    mod = layout_and_allocate(info, flags);
    ......
    err = find_module_sections(mod, info);
    err = check_module_license_and_version(mod);
    setup_modinfo(mod, info);
    ......
    err = complete_formation(mod, info);
    ......
    err = mod_sysfs_setup(mod, info, mod->kp, mod->num_kp);
    ......
    return do_init_module(mod);
}
```

do_init_module 的核心操作就是回调 init：
``` C
static noinline int do_init_module(struct module *mod)
{
    ......
    if (mod->init != NULL)
        ret = do_one_initcall(mod->init);
    ......
    return 0;
}

int __init_or_module do_one_initcall(initcall_t fn)
{
    ......
    if (initcall_debug)
        ret = do_one_initcall_debug(fn);
    else
        ret = fn();
    ......
    return ret;
}
```

#### 模块卸载

模块卸载的操作由 delete_module 来完成，位于 linux3.5.2/kernel/module.c 中。通过回调 exit 完成模块的出口函数功能，最后调用 free_module 将模块卸载。
``` C
SYSCALL_DEFINE2(delete_module, const char __user *,
        name_user, unsigned int, flags)
{
    struct module *mod;
    char name[MODULE_NAME_LEN];
    int ret, forced = 0;
    ......
    if (mod->exit != NULL)
        mod->exit();
    ......
    free_module(mod);
    ......
}        
```

内核模块其实并不神秘。传统的用户程序需要编译为可执行程序才能执行，而模块程序只需要编译为目标文件的形式便可以加载到内核，有内核实现模块的链接，将之转化为可执行代码。同时，在内核加载和卸载的过程中，会通过函数回调用户定义的模块入口函数和模块出口函数，实现相应的功能。


----------


## Linux 内核块设备介绍

#### 块设备概念

一种具有一定结构的随机存取设备，对这种设备的读写是按块进行的，他使用缓冲区来存放暂时的数据，待条件成熟后，从缓存一次性写入设备或者从设备一次性读到缓冲区。可以随机访问，块设备的访问位置必须能够在介质的不同区间前后移动。

#### 块设备相关属性

- 扇区(Sectors)：任何块设备硬件对数据处理的基本单位。通常，1 个扇区的大小为 512byte。（对设备而言）
- 块  (Blocks)：由 Linux 制定对内核或文件系统等数据处理的基本单位。通常，1 个块由 1 个或多个扇区组成。（对 Linux 操作系统而言）
- 段(Segments)：由若干个相邻的块组成。是 Linux 内存管理机制中一个内存页或者内存页的一部分。
!!!
<center>
!!!
![20181130202335.jpg][1]
!!!
</center>
!!!

#### 块设备被访问的分层实现

首先块设备驱动是以何种方式对块设备进行访问的。在 Linux 中，驱动对块设备的输入或输出 (I/O) 操作，都会向块设备发出一个请求，在驱动中用 request 结构体描述。但对于一些磁盘设备而言请求的速度很慢，这时候内核就提供一种队列的机制把这些 I/O 请求添加到队列中(即：请求队列)，在驱动中用 request_queue 结构体描述。在向块设备提交这些请求前内核会先执行请求的合并和排序预操作，以提高访问的效率，然后再由内核中的 I/O 调度程序子系统(即：上图中的 I/O 调度层)来负责提交 I/O 请求，I/O 调度程序将磁盘资源分配给系统中所有挂起的块 I/O 请求，其工作是管理块设备的请求队列，决定队列中的请求的排列顺序以及什么时候派发请求到设备，关于更多详细的 I/O 调度知识这里就不深加研究了。

然后块设备驱动又是怎样维持一个 I/O 请求在上层文件系统与底层物理磁盘之间的关系呢？这就是上图中通用块层 (Generic Block Layer) 要做的事情了。在通用块层中，通常用一个 bio 结构体来对应一个 I/O 请求，它代表了正在活动的以段 (Segment) 链表形式组织的块 IO 操作，对于它所需要的所有段又用 bio_vec 结构体表示。

块设备驱动又是怎样对底层物理磁盘进行反问的呢？上面讲的都是对上层的访问对上层的关系。Linux 提供了一个 gendisk 数据结构体，用他来表示一个独立的磁盘设备或分区。在 gendisk 中有一个类似字符设备中 file_operations 的硬件操作结构指针，他就是 block_device_operations 结构体。
!!!
<center>
!!!
![20181130202336.jpg][2]
!!!
</center>
!!!

#### 块设备数据结构

- block_device：block_device 结构代表了内核中的一个块设备。它可以表示整个磁盘或一个特定的分区。当这个结构代表一个分区时，它的 bd_contains 成员指向包含这个分区的设备，bd_part 成员指向设备的分区结构。当这个结构代表一个块设备时， bd_disk 成员指向设备的 gendisk 结构。
- gendisk 是一个单独的磁盘驱动器的内核表示。内核还使用 gendisk 来表示分区。
- block_device_operations 结构是块设备对应的操作接口，是连接抽象的块设备操作与具体块设备操作之间的枢纽。

系统对块设备进行读写操作时，通过块设备通用的读写操作函数将一个请求保存在该设备的操作请求队列（request queue）中，然后调用这个块设备的底层处理函数，对请求队列中的操作请求进行逐一执行。request_queue 结构描述了块设备的请求队列。
``` C
struct block_device {
    dev_t           bd_dev;
    struct inode   *bd_inode; 		// 分区节点
    int             bd_openers;
    struct semaphore bd_sem;        	// 打开/关闭锁
    struct semaphore bd_mount_sem;  	// 加载互斥锁
    struct list_head bd_inodes;
    void       	*bd_holder;
    int        	bd_holders;
    unsigned   	bd_block_size;     	// 分区块大小
    struct block_device 	*bd_contains;
    struct hd_struct    	*bd_part;
    struct gendisk      	*bd_disk;
    struct list_head    	bd_list;
    unsigned 	bd_part_count; 		// 打开次数
    int 		bd_invalidated;
    struct backing_dev_info *bd_inode_backing_dev_info;
    unsigned long bd_private;
};
```

向内核注册和注销一个块设备可使用如下函数：
``` C
int register_blkdev(unsigned int major, const char *name);
int unregister_blkdev(unsigned int major, const char *name);
```

#### 块设备驱动开发

1. 块设备驱动注册与注销
块设备驱动中的第 1 个工作通常是注册它们自己到内核，完成这个任务的函数是 register_blkdev()，其原型为：
``` C
int register_blkdev(unsigned int major, const char *name);
```
与 register_blkdev() 对应的注销函数是 unregister_blkdev()，其原型为：
``` C
int unregister_blkdev(unsigned int major, const char *name);
```
这里，传递给 register_blkdev() 的参数必须与传递给 register_blkdev() 的参数匹配，否则这个函数返回 -EINVAL。
2. 块设备的请求队列操作
标准的请求处理程序能排序请求，并合并相邻的请求，如果一个块设备希望使用标准的请求处理程序，那它必须调用函数 blk_init_queue 来初始化请求队列。当处理在队列上的请求时，必须持有队列自旋锁。初始化请求队列：
``` C
req_queue_t *blk_init_queue(request_fn_proc *rfn, spinlock_t *lock);
```
该函数的第 1 个参数是请求处理函数的指针，第 2 个参数是控制访问队列权限的自旋锁，这个函数会发生内存分配的行为，故它可能会失败，函数调用成功时，它返回指向初始化请求队列的指针，否则，返回 NULL。这个函数一般在块设备驱动的模块加载函数中调用。清除请求队列：
``` C
void blk_cleanup_queue(request_queue_t * q);
```
这个函数完成将请求队列返回给系统的任务，一般在块设备驱动模块卸载函数中调用。
3. 提取请求
``` C
struct request *elv_next_request(request_queue_t *queue);
```
上述函数用于返回下一个要处理的请求（由 I/O 调度器决定），如果没有请求则返回 NULL。
4. 去除请求
``` C
void blkdev_dequeue_request(struct request *req);
```
上述函数从队列中去除1个请求。如果驱动中同时从同一个队列中操作了多个请求，它必须以这样的方式将它们从队列中去除。
5. 分配“请求队列”
``` C
request_queue_t *blk_alloc_queue(int gfp_mask);
```
对于 FLASH、RAM 盘等完全随机访问的非机械设备，并不需要进行复杂的 I/O 调度，这个时候，应该使用上述函数分配 1 个“请求队列”，并使用如下函数来绑定“请求队列”和“制造请求”函数。
``` C
void blk_queue_make_request(request_queue_t * q, make_request_fn * mfn);
void blk_queue_hardsect_size(request_queue_t *queue, unsigned short max);
```
该函数用于告知内核块设备硬件扇区的大小，所有由内核产生的请求都是这个大小的倍数并且被正确对界。但是，内核块设备层和驱动之间的通信还是以 512 字节扇区为单位进行。
6. 步骤
在块设备驱动的模块加载函数中通常需要完成如下工作：
 - 分配、初始化请求队列，绑定请求队列和请求函数。
 - 分配、初始化 gendisk，给 gendisk 的 major、fops、queue等成员赋值，最后添加 gendisk。
 - 注册块设备驱动。

在块设备驱动的模块卸载函数中通常需要与模块加载函数相反的工作：
 - 清除请求队列。
 - 删除 gendisk 和对 gendisk 的引用。
 - 删除对块设备的引用，注销块设备驱动。


----------


## 注册块设备的基本步骤

内核提供了块设备的相关接口（位于源码文件 /block）。一般情况下，需要进行如下步骤：
1. 注册一种块设备类型 register_blkdev，传入的参数是要注册的主设备号和设备名称，如果为 0 的话内核会自动为你分配一个主设备号。
2. 申请一个块设备结构体的实例 alloc_disk。
3. 对块设备实例进行初始化赋值。
4. 将实例插入到内核块设备链表上 add_disk。


----------


## 编写模块源码

#### 模块的注册

模块的注册和卸载通过 module_init(\*fn) 和 module_exit(\*fn) 进行，其中 \*fn 是函数指针，是用于执行初始化或者卸载模块时的具体操作。一个基本的简单模块如下：
``` C
module_init(darkelf_init);
module_exit(darkelf_exit);
```

这里的 darkelf_init 是自己定义的对该模块进行初始化的函数；darkelf_exit 是对本模块进行卸载时用于处理释放内存等操作的函数。此外还可以指定对模块的描述信息：
``` C
MODULE_AUTHOR("XiongXiong <xx@xxiong.me>");
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("You got a dream...you gotta protect it!!");
```

- MODULE_AUTHOR指定本模块的作者，一般是名字+邮箱的格式；
- MODULE_LICENSE是声明本模块的开源协议；
- MODULE_DESCRIPTION是对模块功能的描述信息。

#### 模块的基本数据结构

一个模块一般都有个核心的数据结构，这里我们也自己定义一个结构体，结构体中包含一个 kobject 指针，通过 kobject 可以将模块插入内核维护的 kset 链表中，通过这个结构体，我们可以实现将模块中的函数映射到 sys 文件系统中，从而实现通过在用户空间对文件的读写来间接实现控制模块访问内核地址空间等骚操作：
``` C
struct darkelf {
    struct kobject *kobj;
} *darkelf;
```

#### 初始化函数

初始化函数中一般要执行为变量分配内存空间、将 kobject 插入内核等操作。如下的例子中只完成了为 darkelf 变量分配空间，并将 darkelf 的 kobject 插入 kernel_kobj 的操作。
``` C
static int __init darkelf_init(void)
{
    int ret = 0;
    darkelf = kzalloc(sizeof(struct darkelf), GFP_KERNEL);
    if (!darkelf) {
        ret = -ENOMEM;
        goto out;
    }
    darkelf->kobj = kobject_create_and_add("darkelf", kernel_kobj);
    if (!(darkelf->kobj)) {
        ret = -ENOMEM;
        goto free_darkelf;
    }
    return ret;
free_darkelf:
    kfree(darkelf);
out:
    return ret;
}
```

#### 注销函数

注销模块一般是删除模块信息、回收内存等操作。
``` C
static void __exit darkelf_exit(void)
{
    kobject_put(darkelf->kobj);
    kfree(darkelf);
}
```

#### 注册 sysfs 文件节点

内核通过 sysfs 来实现将模块中的函数映射到一个文件上，从而实现对模块的读写。内核提供了函数 sysfs_create_group() 来实现映射，通过 sysfs_remove_group() 删除映射，具体过程如下。

首先我们随便定义一个变量b，用来存储用户空间传来的值：
``` C
static int b;
```

然后定义 kobj_attribute，并指定对变量 b 的操作函数：
``` C
static struct kobj_attribute darkelf_gate =
       __ATTR(gate, 0664, darkelf_show, darkelf_store);
```

其中 0664 表示将 darkelf_b 映射到 sysfs 形成文件后的访问权限，文件的读写分别映射到函数：darkelf_show 和 darkelf_store上。通过这种方式，在用户空间对该文件执行 echo 或 cat 后，具体的读写都将由这两个函数完成。读写函数可以随便操作，例如下图中，show 执行的操作是将 b 中的值打印出来；store 执行的操作是将用户空间发来的数据转换成 10 进制的 int 类型然后保存到b中：
``` C
static ssize_t darkelf_show(struct kobject *kobj, 
        struct kobj_attribute *attr, char *buf)
{
    return 0;
}

static ssize_t darkelf_store(struct kobject *kobj, 
        struct kobj_attribute *attr, const char *buf, size_t cnt)
{
    int ret = 0;
    ret = kstrtoint(buf, 10, &b);
    if (ret < 0)
        return ret;
    return ret;
}
```

再然后定义 attributes 和 attribute_group，如下的例子：
``` C
static struct attribute *darkelf_attrs[] = {
    &darkelf_gate.attr,
    NULL,
};

static struct attribute_group darkelf_attr_group = {
    .attrs = darkelf_attrs,
};
```

准备工作完成后，只需要在 init 和 exit 中分别将创建和删除的函数插入，就完成初始化过程了。


----------


## 编写 Makefile
``` C
obj-m := darkelf.o
CURRENT_PATH := $(shell pwd)
LINUX_KERNEL_PATH := /root/linux-4.4.126
all:
        $(MAKE) -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) modules
```

首先第一行指定将模块编译成独立模块的形式，第三行获取当前的路径，第四行是内核源码的路径。通过这种方式，可以在不用反复编译内核的情况下，将模块编译出来。make 将自动从内核所在路径中加载需要的头文件。通过执行 make all 来完成编译：
``` bash
root@AE86_<darkelf>(lab1)make all
make -C /root/linux-4.4.126 M=/root/darkelf modules
make[1]: Entering directory '/root/linux-4.4.126'
  CC [M]  /root/darkelf/darkelf.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /root/darkelf/darkelf.mod.o
  LD [M]  /root/darkelf/darkelf.ko
make[1]: Leaving directory '/root/linux-4.4.126'
```

然后当前路径下将出现一个 darkelf.ko，这就是模块文件。
``` bash
root@AE86_<darkelf>(lab1)ls -lh
total 688K
-rw-r--r-- 1 root root 3.8K Apr 25 15:54 darkelf.c
-rw-r--r-- 1 root root 329K Apr 25 17:08 darkelf.ko
-rw-r--r-- 1 root root  561 Apr 25 17:08 darkelf.mod.c
-rw-r--r-- 1 root root  93K Apr 25 17:08 darkelf.mod.o
-rw-r--r-- 1 root root 241K Apr 25 17:08 darkelf.o
-rw-r--r-- 1 root root  156 Apr  3 19:26 Makefile
-rw-r--r-- 1 root root   32 Apr 25 17:08 modules.order
-rw-r--r-- 1 root root    0 Apr  3 19:31 Module.symvers
```


----------


## 执行编译并测试模块

通过 insmod 将模块插入内核中，通过 rmmod 将模块弹出：
``` bash
root@AE86_<darkelf>(lab1)insmod ./darkelf.ko
root@AE86_<darkelf>(lab1)rmmod darkelf
```

进入 /sys/kernel 中，会发现有一个名为 darkelf 的目录，该目录下有一个文件 b：
``` bash
root@AE86_<darkelf>ll
total 0
-rw-r--r-- 1 root root 4K Apr 25 17:17 b
root@AE86_<darkelf>pwd
/sys/kernel/darkelf
```

测试对文件的读写：
``` bash
root@AE86_<darkelf>cat b
0
root@AE86_<darkelf>echo 12345 > b & cat b
12345
```

查看调试信息：
``` bash
root@AE86_<darkelf>dmesg | tail -n 2
[  156.351914] DARKELF: ghost has fall down.
[  159.732729] DARKELF: dark elf has gone away.
```

查看模块 .ko 的信息：
``` bash
root@AE86_<darkelf>(lab1)readelf -h darkelf.ko
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          322160 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         29
  Section header string table index: 28
```


----------


## 编写块设备源码

定义一个结构体，其中包含一个 gendisk 指针和请求队列指针
``` C
struct darkelf {
    struct kobject *kobj;
    struct gendisk *disk;
    struct request_queue *queue;
} *darkelf;
```

初始化一个对当前块设备的操作方法
``` C
static const struct block_device_operations elf_fops = {
    .owner    = THIS_MODULE,
    .open     = elf_open,
    .release  = elf_release,
};
```

这里先让 elf_open 和 elf_release 两个函数返回空值
``` C
static int elf_open(struct block_device *bdev, fmode_t mode)
{
     return 0;
}

static void elf_release(struct gendisk *disk, fmode_t mode) { }
```

通过 register_blkdev 注册一个新的设备类型
``` C
elf_major = register_blkdev(0, "darkelf");
```

初始化块设备和请求队列
``` C
darkelf->disk = alloc_disk(16);
if (!darkelf->disk) {
        ret = -ENOMEM;
        goto free_group;
}
darkelf->queue = blk_alloc_queue_node(GFP_KERNEL, NUMA_NO_NODE);
if (!darkelf->queue) {
        ret = -ENOMEM;
        goto free_disk;
}
```

对请求队列和块设备实例进行初始化
``` C
blk_queue_make_request(darkelf->queue, elf_queue_bio);
BUG_ON(!darkelf->queue);
darkelf->queue->nr_queues = 1;
darkelf->queue->queuedata = darkelf;

queue_flag_set_unlocked(QUEUE_FLAG_NONROT, darkelf->queue);
queue_flag_clear_unlocked(QUEUE_FLAG_ADD_RANDOM, darkelf->queue);
blk_queue_logical_block_size(darkelf->queue, 512);
blk_queue_physical_block_size(darkelf->queue, 512);

strncpy(darkelf->disk->disk_name, "darkelf", DISK_NAME_LEN);
darkelf->disk->flags = GENHD_FL_EXT_DEVT;
darkelf->disk->major = elf_major;
darkelf->disk->first_minor = 7;
darkelf->disk->fops = &elf_fops;
darkelf->disk->queue = darkelf->queue;
darkelf->disk->private_data = (void*)darkelf;
darkelf->queue->queuedata = (void*)darkelf;
set_capacity(darkelf->disk, 1024);
```

通过 add_disk 将块设备实例插入内核
``` C
add_disk(darkelf->disk);
```


----------


## 编译并测试块设备

编译所需要的 Makefile 采用实验二的 Makefile，执行 make 后，将生成 darkelf.ko，然后通过 insmod darkelf.ko 将该模块插入内核中。首先查看 /dev 下面我们插进去的模块：
``` bash
root@AE86_<dev>ls -lah /dev | grep darkelf
brw-------  1 root root      249,   0 Apr 25 17:15 darkelf
```

通过 lsblk 查看块设备：
``` bash
root@AE86_<dev>lsblk
NAME     MAJ:MIN 	RM 	SIZE 	RO 	TYPE 	MOUNTPOINT
sda        8:0    	0  	50G  	0 	disk
└─sda1     8:1   	0  	50G  	0 	part 	/
darkelf  249:7		0	512K	0	disk
```


----------


## 答疑补充

#### Makefile 编写的注意事项
!!!
<center>
!!!
![20181130202337.jpg][3]
!!!
</center>
!!!

#### 模块验证失败警告的处理
``` bash
[   276.204699] darkelf: module verification failed: signature and/or required ....
```

Makefile 中加入下面语句关闭验证
``` bash
CONFIG_MODULE_SIG=n
```

#### 文件内核版本与当前内核版本不一致的问题

1. 内核版本不一样
 - 切换当前内核版本为源码的内核版本
 - 或将链接的内核源码改成与当前内核版本相同的源码
2. 内核版本相同，但只人为修改了版本号
 - 将源码版本号修改回来再编译

例如：linux-4.4.128-20154043 修改回 linux-4.4.128

#### 编译问题————在 Ubuntu 下编译内核的各种依赖问题

1. 安装 ncurses 库失败
 - 首先，安装 ncurses 库的目的————使用 make menuconfig 时以可视化的方式进行编译选项的选择。事实上这种方式很鸡肋。解决方法：不用安装 ncurses 库，使用 make oldconfig 进行编译选项的选择，配合当前系统中已有的 .config，可以直接一路回车键默认选过去。需要修改某个模块的编译选项时，可以直接用编辑器编辑 .config 文件。
2. 使用 make-kpkg 工具
 - 不需要安装这个工具，直接：
``` bash
make modules_install & make install
```
 - 若要清理中间文件，可以直接：
``` bash
make clean
```
3. 修改 grub 文件
 - 不需要

#### 一图流编译总结！！！
!!!
<center>
!!!
![20181130202338.jpg][4]
!!!
</center>
!!!

（完）


  [1]: http://blog.xxiong.me/usr/uploads/2018/12/2374434949.jpg
  [2]: http://blog.xxiong.me/usr/uploads/2018/12/1156424870.jpg
  [3]: http://blog.xxiong.me/usr/uploads/2018/12/1733982065.jpg
  [4]: http://blog.xxiong.me/usr/uploads/2018/12/1820028190.jpg
