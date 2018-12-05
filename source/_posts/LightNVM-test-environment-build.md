---
title: LightNVM 测试环境搭建
date: 2018-12-05 11:12:03
tags:
---


要使用 Open-Channel SSD，需要得到操作系统内核支持。 随着 LightNVM 的加入，4.4 版本后的 linux 内核都可以支持。该项目仍然处于开发中，最新的源代码可从 [https://github.com/OpenChannelSSD/linux][1] 获得。启用了相应的内核支持后，必须满足以下条件：
 - 兼容的设备，如 QEMU NVMe 或 Open-Channel SSD，如 CNEXLabs LightNVM SDK；
 - 设备驱动程序顶部的媒体管理器。 媒体管理器管理设备的分区表；
 - 在 Block Manager 上层暴露出 Open-Channel SSD 的目标（target）；


----------


## 实验环境

目前由于 CNEXLabs 的 Open-Channel SSD 还没有购置。因此利用 Qemu 虚拟机模拟 Open-Channel SSD 设备。注意 Qemu 必须能够支持 Open-Channel，可以使用 CNEXLabs 提供的 [Qemu-nvme 分支][2]。依照官方文档进行配置和编译后，才可以通过后端文件暴露出一个 LightNVM 兼容的设备。当完成安装，内核也被顺利启动后，设备就能够被观察到，并且可以通过 nvme-cli 的工具进行管理和初始化。实验内核：linux-4.12-rc2。


----------


## 实验设计

#### lightnvm hello world

环境配置成功后，可以首先重复一下论文中的一些实验进行验证，观察和分析 OCSSD 的特点，熟悉 lightnvm 的使用。

#### 多租户应用实验

利用 OCSSD 来实现多租户的应用，每一个 target 对应一个租户，以同样的 workload，与传统的 SSD 的数据进行对比，分析结果。改变 workload 后再同样进行分析，发现 lightnvm 的特点和不足。

#### 算法改进

由于 lightnvm 把 FTL 的功能等都提高到上层，数据迁移，GC 等算法都能够看到如何被实现，也意味着可以进行优化，并且编译自己改进的内核，从而有针对性地提高性能。


----------


## 配置环境参考

下面介绍目前我们能成功跑通 LightNVM 例程的环境搭建流程，主机端安装的是 Ubuntu 17.04 发行版，内核版本为 4.10，提供编译内核的环境，还存在一些不便和不足，后期继续优化：

#### 内核版本 linux-4.12-rc2 的编译

 - 从 GitHub 下载 [master 分支][3]；

 - 编译内核，需要注意配置文件中，应该确定以下被配置：
``` bash
CONFIG_NVM=y 
CONFIG_NVM_DEBUG=y 
CONFIG_NVM_PBLK=y 
CONFIG_BLK_DEV_NVME=y
```

 - 在 arch/x86/boot/ 目录下，会生成编译好后的 bzImage 文件，大小 7M 左右；

#### Qemu的编译

 - 从 GitHub 下载 [Qemu 主分支][4]；

 - 运行如下的配置命令（官方文档中也有详细的介绍）：
``` bash
./configure --enable-linux-aio --target-list=x86_64-softmmu --enable-kvm
modprobe kvm-intel # 安装 kvm 模块
```
 - 编译的过程中可能需要安装一些库

#### Qemu 运行虚拟机

 - 创建一个空的文件来模拟 nvme 的设备：
``` bash
dd if=/dev/zero of=blknvme bs=1M count=1024
```

 - 用以下命令启动一个预装的 linux 镜像作为 Qemu 的运行环境：
``` bash
qemu-system-x86_64 -m 4G -smp 1 --enable-kvm -hda $LINUXVMFILE -append \
    "root=/dev/sda1" -kernel "/home/foobar/git/linux/arch/x86_64/boot/bzImage" \
    -drive file=blknvme, if=none, id=mynvme -device nvme, drive=mynvme, \
    serial=deadbeef, namespaces=1, lver=1, nlbaf=5, lba_index=3, mdts=10
```

 - 其中，用预装好的镜像 .img 文件替换 $LINUXVMFILE，我们采用的是 Ubuntu 16.04 的系统。可以在 Qemu 里安装，也可以在 virtual-box 等虚拟机中装好，再通过下面的命令加载：
``` bash
qemu-img convert -f vmdk -O raw ubuntu.vmdk image.img
```

#### Hello world

 - 安装 nvme-cli 工具，通过 apt install 或者从 [GitHub][5] 下载；

 - 一切就绪后，就可以通过 sudo nvme lnvm list 命令来列出设备；

（完）

  [1]: https://github.com/OpenChannelSSD/linux
  [2]: https://github.com/OpenChannelSSD/qemu-nvme
  [3]: http://github.com/OpenChannelSSD/linux
  [4]: https://github.com/OpenChannelSSD/qemu-nvme.git
  [5]: https://github.com/linux-nvme/nvme-cli
