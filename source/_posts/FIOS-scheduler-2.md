---
title: FIOS 调度器实现
date: 2018-12-05 11:24:09
tags:
---


## 核心数据结构分析
fios 队列结构：
``` C
struct fios_queue {
    pid_t pid;                  // 进程的pid
    struct rb_node fiosq_node;  // 将fios queue插入到fios rb-tree，主要用于在dispatch阶段，快速的取出下一个要被处理的fios queue
    struct rb_node pid_node;    // 将fios queue插入到pid rb-tree，主要用于在插入请求add request阶段，能够快速的通过进程pid找到要插入的fios queue
    struct rb_root rq_list;     // 组织请求
    struct list_head fifo;      // 为了确保不会出现请求饥饿
```

fios 调度器实例结构
``` C
struct fios_data {
    struct request_queue *queue;     // 指向驱动的queue，dispatch阶段时插入
    struct rb_root fiosq_list;       // 用于dispatch时快速查找fios queue
    struct rb_root pid_list;         // 用于add request阶段快速查找fios queue
    struct fios_queue *active_fiosq; // 当前正在进行dispatch的fios queue
    struct fios_queue async_fiosq;   // 存放异步请求的fios queue
    struct fios_queue oom_fiosq;     // 当无法分配新queue时，将请求放入其中
    int fios_alpha;                  // IO anticipation的参数
    u64 fios_tsrv;                   // IO anticipation的参数
    u64 fios_tsrv_nr;                // IO anticipation的参数
    u64 fios_tsrv_start;
    u64 fios_tsrv_sum;
    bool fios_on_anti;               // 用于标记当前是否正处于anticipation的状态中
```


----------


## add request 的过程
!!!
<center>
!!!
![20181130202363.jpg][1]
!!!
</center>
!!!


----------


## dispatch 的过程
!!!
<center>
!!!
![20181130202364.jpg][2]
!!!
</center>
!!!


----------


## trace 的相关补充

1. 切换调度器
``` bash
echo "fios" > /sys/block/sda/queue/scheduler
```
2. 查看调度器
``` bash
cat /sys/block/sda/queue/scheduler
```
3. 随机读
``` bash
fio -filename=/tmp/test_randread -direct=1 -iodepth 1 -thread \
    -rw=randread -ioengine=psync -bs=16k -size=2G -numjobs=10 \
    -runtime=60 -group_reporting -name=mytest
```
4. 顺序读
``` bash
fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=read \
    -ioengine=psync -bs=16k -size=2G -numjobs=10 -runtime=60 \
    -group_reporting -name=mytest
```
5. 随机写
``` bash
fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread \
    -rw=randwrite -ioengine=psync -bs=16k -size=2G \
    -numjobs=10 -runtime=60 -group_reporting -name=mytest
```
6. 顺序写
``` bash
fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread \
    -rw=write -ioengine=psync -bs=16k -size=2G -numjobs=10 \
    -runtime=60 -group_reporting -name=mytest
```
7. 混合随机读写
``` bash
fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=randrw \
    -rwmixread=70 -ioengine=psync -bs=16k -size=2G -numjobs=10 \
    -runtime=60 -group_reporting -name=mytest -ioscheduler=noop
```


----------


## 相关说明

#### 参数
| 参数 | 参数解释 |
| ------ | ------ |
| filename=/dev/sdb1 | 测试文件名称，通常选择需要测试的盘的data目录。 |
| direct=1 | 测试过程绕过机器自带的buffer。使测试结果更真实。 |
| rw=randwrite | 测试随机写的I/O |
| rw=randrw | 测试随机写和读的I/O |
| bs=16k | 单次io的块文件大小为16k |
| bsrange=512-2048 | 同上，提定数据块的大小范围 |
| size=5g | 本次的测试文件大小为5g，以每次4k的io进行测试。 |
| numjobs=30 | 本次的测试线程为30. |
| runtime=1000 | 测试时间为1000秒，如果不写则一直将5g文件分4k每次写完为止。 |
| ioengine=psync | io引擎使用pync方式 |
| rwmixwrite=30 | 在混合读写的模式下，写占30% |
| group_reporting | 关于显示结果的，汇总每个进程的信息。 |
| lockmem=1g | 只使用1g内存进行测试。 |
| zero_buffers | 用0初始化系统buffer。 |
| nrfiles=8 | 每个进程生成文件的数量。 |

#### trace 的读写模式
| 模式 | 模式解释 |
| ------ | ------ |
| read | 顺序读 |
| write | 顺序写 |
| rw,readwrite | 顺序混合读写 |
| randwrite | 随机写 |
| randread | 随机读 |
| randrw | 随机混合读写 |

#### fio 测试的输出结果
| 输出 | 输出解释 |
| ------ | ------ |
| io | 总的输入输出量 |
| bw | 带宽 KB/s |
| iops | 每秒钟的IO数 |
| runt | 总运行时间 |
| lat (msec) | 延迟(毫秒) |
| msec | 毫秒 |
| usec | 微秒 |

（完）


  [1]: http://blog.xxiong.me/usr/uploads/2018/12/4038320473.jpg
  [2]: http://blog.xxiong.me/usr/uploads/2018/12/2168694059.jpg
