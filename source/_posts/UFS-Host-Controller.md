---
title: UFS Host Controller 工作流程
date: 2018-12-05 11:17:52
tags:
---


该文档描述了 UFS Host Controller 的主要运作流程以及 qemu 中，host controller 的接口函数设计。该文档的内容均参考自 JEDEC STANDARD JESD223C 标准配置文档以及 qemu 中设备模拟源代码。


----------


## UFS 架构图

JESD223C 为 UFS 设备定义了一个通用的 host controller interface（HCI）, 主机的驱动程序通过与 HCI 工作，达到与 UFS 设备进行交互的目的。该标准定义了寄存器级的 HCI 设计，负责主机 software 和 UFS 设备间的接口管理和数据传输。如下架构图：
!!!
<center>
!!!
![20181130202321.jpg][1]
!!!
</center>
!!!

蓝色双向箭头上方的为主机软件部分，下方为硬件部分（host controller 以及 UFS 设备）。该项目中使用 Qemu 进行模拟的代码主要为实现下图中 UFS Host Controller Interface 的功能。

#### 接口架构

主机的 software 通过内存中的一系列 host registers 和一些称为 Transfer Request Descriptors 的数据结构来与 Host controller 硬件进行交互。UFSHCI 定义了两种接口空间。MIMO Space 以及 Host Memory Space，下图展示除了 UFS HCI 的概念框图。
!!!
<center>
!!!
![20181130202322.jpg][2]
!!!
</center>
!!!

#### MMIO Space
在这个空间中，hardware register 被定义为 host software 的接口，通过 MMIO 的方式被实现，主要包含了下面三种类型的寄存器：
1. Host Controller Capability Registers. 这些寄存器提供了关于 Host controller 功能的描述
2. Runtime and Operation Registers，包括对以下几种功能的支持。
 - Interrupt configuration. 这些寄存器为 host software 提供了使能/中止以及中断状态的接口。
 - Host controller status. 该寄存器显示 host controller 的状态，并允许主机 software 初始化/禁用 host controller
 - UTP transfer Request List management. 这些寄存器给 UTP Transfer Request List 提供了接口。
 - UTP Task Management request lists management. 这些寄存器为 UTP Task Management request list 提供了一个接口。
 - UIC Command Registers。这些寄存器为 Unipro 配置以及控制提供了接口。
3. Vendor Specific Registers. 由供应商进行定义。

#### Host Memory Space

该空间中包含了能够描述将执行命令和命令中数据缓存的数据结构。简单地可以理解为：UTP Transfer Request List 管理通用的 IO 命令，而 UTP Task Mangement Request List 对应管理命令。


----------


## 传输请求接口（Transfer Request Interface）

一个 UFS 的 host systerm 由一些硬件和软件层组成（host controller 和 host software）。下图展示了其层次结构。
!!!
<center>
!!!
![20181130202323.jpg][3]
!!!
</center>
!!!

前面定义的数据结构和寄存器就是为了实现上图中的这 4 个 service access points(SAPs) 而定义。分别为 UIO_SAP, UDM_SAP, UTP_CMD, UTP_TM_SAP。为了管理 host software 和 UFS 设备间的通信，host controller 为主机 software 发送出传输请求提供了如下 3 个独立的接口。
1. UTP Transfer Request List. 该 List 由一系列名为 Transfer Request Descriptor(UTRD) 的数据结构构成，最多为 32 个。UTRD 详细描述了一个即将被执行的命令以及相关的数据。UFS 的主机 software 通过将一个 UTRD 插入这个 list 并且提示 Host controller doorbell 来发起一个命令。然后，命令就会按在 list 中的顺序被分派，但是被执行完成的顺序可能会改变。此时，host controller 就代表处理器管理与这个命令有关的所有数据传输的操作。命令完成将产生中断或者更新 UTRD 中的状态字段。
2. UTP Task Management Request List. 该 list 包含了名为 UFS Task Management Request Descriptor（UTMRD）的数据结构。UTMRD 详细描述了 host software 希望设备执行的任务管理函数。所有的 Task Management Request 将优先于上面 UTP Transfer Request 的执行，两者的处理流程类似。
3. UIC Command Register. 这个寄存器集是被主机 software 直接用来执行一个 UIC 命令。


----------


## UFS Transport Protocol （UTP）Layer 简述

host driver 与 UTP 层直接联系，而两者的交流是被分为一系列的消息（messages），这些消息根据标准组织为 UFS Protocol Information Units(UPIU) 的格式。UFS2.1 文档中的表 10-2 解释了不同种类型的 UPIU 的描述，比如 NOP Out, Command, Response，Data Out 等类型。UPIU 的内容是存放在名为 UTP Command Descriptor（UCD）的数据结构中，而前面所介绍的 UTRD（UTP Transfer Request descriptor）中存放有指向 UCD 的指针。

#### 一个数据传输命令的示例

数据在 host memory, host controller, device 三者之间都可以理解为以 UPIU 的形式来进行传递。下图给出了一个从 host 端到 device 端传输 577 byte 数据的例子：
!!!
<center>
!!!
![20181130202331.png][4]
!!!
</center>
!!!

可以假设这是一个写 577 byte 到设备上的命令。由 Command UPIU 开始，告知所需要写的数据信息，收到 RTT UPIU（ready to transfer），表示 device 可以接受数据传输，通过 DATA OUT UPIU 进行数据的传输。可以看到途中 Data OUT UPIU 的顺序可以改变的。最终收到 Response UPIU 表示这一次写命令的结束。其他命令的执行过程类似。


----------


## Host software 与 Host controller 的具体交互流程

根据标准中的描述，Software 对于 UFS HCI 的操作分为三大部分：Host controller 配置和控制，数据传输操作，任务任务管理。下面部分中所提及到的寄存器及其字段都能够在 UFS HCI2.1 文档中第 5 章中查到。

#### Host Controller 初始化

当主机控制器出现上电复位时，所有 MMIO 寄存器都将处于其上电默认状态，并且链路将处于非活动状态（inactive）。 以下是主机软件将执行的操作序列以初始化 host controller：

1. 启动 host controller 的第一步是正确编程系统总线接口。 这一步与控制器实现所使用的系统总线有关，因此应遵循特定系统总线的文档。该步完成时，控制器应准备好在系统总线上传输数据。
2. 为了启用主机控制器，将 1 写入 HCE(host controller enable) 寄存器。 这触发了本地 UIC 层的自主基本初始化(autonomous basic initialization)。 初始化序列应由 DME_RESET 和 DME_ENABLE 命令组成。 可以根据实现需要添加其他命令，如 DME_SET 命令。 在基本初始化序列期间，HCE 的值一直为 0。
3. 等待 HCE 读为 1 的时候继续。 表示基本的初始化序列已经完成。
4. 附加命令（如 DME_SET 命令）可以从系统主机发送到 UFS 主机控制器，以提供配置灵活性。
5. 可选地将 IE.UCCE(UIC command completion enable)设置为 1，以便使能 IS.UCCS(UIC Command Completion Status)中断。
6. 发送 DME_LINKSTARTUP 命令以开始链路启动过程 (link startup procedure)
7. 完成 DME_LINKSTARTUP 命令设置 IS.UCCS 位.如果设置了 IE.UCCE，则可以向系统主机标记中断, 该中断将独立于 GenericErrorCode 进行标记。
8. 如果完成的 DME_LINKSTARTUP 命令的 GenericErrorCode 为 SUCCESS，那么除了 IS.UCCS 位之外，还将设置 HCS.DP(device present)。
9. 检查 HCS.DP 的值，并确保连接有一个设备。 如果检测到设备的存在，转到步骤 10;否则，在 IS.ULSS 设置为 1 后重新发送 DME_LINKSTARTUP 命令（转到步骤 6）。 IS.ULSS 等于 1 表示 UFS 设备已准备好进行链接启动。
10. 通过编程 IE 寄存器来启用附加中断。
11. 以阈值（IACTH）和超时（IATOVAL）的所需要的值来初始化中断聚合控制寄存器（UTRIACR）。注意当运行/停止寄存器（UTRLRSR）未被使能或没有请求未完成时，也可以随时执行UTRIACR初始化。
12. 如果需要，通过 UIC 命令界面 (interface) 完成主机控制器配置。
13. 分配和初始化 UTP 任务管理请求列表。(Allocate and initalize UTP Task Management Request List)
14. 编程 UTP Task Management Request List 的起始地址，并且用一个 64 位地址指针指向它。
15. 分配和初始化 UTP 传输请求列表（UTP Transfer Request List）
16. 编程 UTP Transfer Request List 的起始地址，并且用一个 64 位地址指针指向它。
17. 通过将 UTP 任务管理请求列表 Run-Stop Register（UTMRLRSR）设置为 1，启用 UTP Task-Mangement Request List。 该操作允许 host controller 通过 UTP Task Management Request Door Bell 机制开始接受 UTP 任务管理请求。
18. 通过将 UTP 传输请求列表 Run-Stop Register（UTRLRSR）设置为 '1' 来启用 UTP Transfer Request List。该操作允许主机控制器通过 UTP Transfer Request Door Bell 机制开始接受 UTP 传输请求。
19. bMaxNumOfRTT 将被设置为 bDeviceRTTCap 和 NORTT 的最小值。

#### 配置与控制

一旦主机控制器复位，主机软件就可以使用 UIC 命令寄存器来配置和控制链路以及连接的设备。 主机软件负责配置 UFS 互连堆栈和连接的设备。 在对 UIC 命令寄存器编程时，只有在设置了所有 UIC 命令参数寄存器（UICCMDARG1，UICCMDARG2 和UICCMDARG3）之后，主机软件才能设置寄存器 UICCMD。 检查执行 UIC 命令的状态有两种选择：
1. 通过中断机制。在将 UIC 命令发送到主机控制器执行之前，软件通过设置 UIC 命令完成启用寄存器 IE.UCCE 来启用 UIC 命令完成中断。命令执行完成后，主机控制器将产生一个中断，并将寄存器设置为 UIC COMMAND 完成状态寄存器 '1'。
2. Software pulling。在将 UIC 命令发送到主机控制器执行之后，软件将持续 pulling UIC 命令完成状态，直到它返回 “1”。
一旦命令完成，软件可以检查返回/状态代码（如果适用于该命令）。

#### CRYPTOCFG 配置过程（与数据加密有关，模拟中可以简化舍弃）

要在 CRYPTOCFG 阵列中配置一个 entry，software 需遵循以下步骤：
1. 在 CRYPTOCFG 数组中选择一个条目 x-CRYPTOCFG
2. 如果条目未启用，比如 x-CRYPTOCFG.CFGE == 0，跳到步骤 5
3. 验证没有待处理的事务在其 CCI 字段中引用 x-CRYPTOCFG，即对于所有挂起的事务，UTRD.CCI≠x。验证完毕后，进入步骤 4
4. 通过将 0000h 写入 x-CRYPTOCFG 的 DW16 清除 x-CRYPTOCFG.CFGE 位。此操作也会清除 CAPIDX 和 DUSIZE 字段
5. 根据以下规则将加密密钥写入 x-CRYPTOCFG.CRYPTOKEY 字段：
 - 按照 6.3 节中列出的算法特定布局组织密钥。 没用过 CRYPTOKEY 的区域应该用零写
 - 密钥以小端格式写入：CRYPTOKEY [0] 的字节 0，字节 1 到 CRYPTOKEY [1]，字节 15 到 CRYPTOKEY [15] 等。
 - CRYPTOKEY 的内容应该从 DW0 到 DW15 顺序地在一个原子操作集中写入
6. 选择写入 x-CRYPTOCFG 的 DW17
7. 用 CAPIDX，DUSIZE和CFGE = 1 写入 x-CRYPTOCFG 的 DW16

只有在上述程序完成之后，软件才可以使用新的配置，通过 UTRD.CCI = x 来发起一个 transaction。


----------


## 创建一个 UTP Transfer Request 的基本步骤

Host controller 复位和配置完成后，software 可以使用 UTP Transfer Request List（UTRL）将 UTP 命令传递到连接到链路的 UFS 设备。UTRL 是位于系统内存中的 list buffer，用于将命令从软件传递到设备。software 负责根据 CAP.NUTRS 的值选择 UTRL 大小。通常，软件应该选择 32 entry 选项，除非系统功能规定了较小的内存占用。当主机软件需要向主机控制器发送 UTP 命令时，它使用 UTP 传输请求列表。 以下是主机 software 构建 UTP 传输请求的步骤。

1. 通过读取 UTRLDBR 寄存器找到一个空的传输请求 slot。An empty transfer request slot 在 UTRLDBR 中的相应位的值为 0。
2. 主机软件在空 slot 中构建 UTRD。
 - 编程字段 CT（command type），用于指示命令类型：SCSI，原生 UFS 命令或设备管理函数
 - 编程 DD 字段，其作为命令的一部分包含了数据操作的方向。
 - 如果软件请求将命令标记为中断命令（IS.UTRCS 在命令完成时设置），则 I（中断）	置 1。 如果软件请求将该命令标记为常规命令，该位将被清除。
 - 用'Fh'初始化 OCS。
 - 分配和初始化 UCD（UTP command descriptor）。
 - 用一个 UTP command 来编程 UPIU Command 字段，在 UCD 中，不包括任务管理功能。
 - 在 UCD 中用'0'初始化字段 response UPIU。
 - 如果需要，填写与数据相关的所有数据缓冲区的指针和 PRDT(Physical Region Descriptal Table) 的大小。
3. 用 UCD(UTP Command Desciptor) 的起始地址编程字段 UCDBA 和 UCDBAU。
4. 用在 UCD 中的 Response UPIU 的 offset 来编程 RUO（Response UPIU Offset）字段。
5. 用 Response UPIU 的长度来编程 RUL 字段（Response UPIU Length）。
6. 如果需要的话，用在 UCD 中的 PRDT 字段的偏移量来编程 PRDTO 字段。
7. 如果需要的话，用 PRDT 的长度来编程 PRDTO 字段。对每一个将要被发送到 host controller 的命令，都要重复如上 1-7 的步骤。
9. 检查寄存器 UTRLRSR（UTP Transfer Request Run-Stop Register），并确保在继续之前读取的值为“1”。
10. 设置 UTP 传输请求中断聚合控制寄存器（UTRIACR）使能位为“1”以使能中断。
11. 将计数器和定时器复位（CTR）位设置为'1'以复位与中断相关的计数器和定时器。
12. 编程字段中断聚合计数器阈值（IACTH）与产生中断所需的命令完成次数。
13. 编程字段中断聚合超时值（IATOVAL）with 允许的响应到达主机控制器之间的最大时间和产生中断。
14. 设置 UTRLDBR 寄存器来 ring the doorbell register 告知 host controller 一个或多个 transfer requests 已经准备好发送给对那个的设备。Host software 应该仅仅写‘1’到对应命令的 bit position。其他在 UTRLDBR 中的位应该被写‘0’，表示当前的值没有改变。


----------


## UPIU的执行过程

#### 由主机 software 产生的 Outbound UPIU

除了 DATA OUT UPIU 之外，所有其他 UFS 定义的 Outbound UPIU 必须通过软件 compose 并存储在 host memory 中。 然后，Host controller 通过 DMA 从 memory 中取出 outbound UPIU，并使用本地 UniPro 堆栈将其分派到 UFS 设备。 UTP engine 仅对 outbound UPIU 的 LUN 和 task tag字段进行分析，以便能够匹配未来相应的inbound UPIU。 注意，对于QUERY REQUEST和NOP OUT UPIU，LUN字段保留，不会用于匹配(因为不需要)。

#### 由Host Controller生成的Outbound UPIU

只有DATA OUT UPIU由UTP engine自动生成，无需software介入。 Data Out UPIU是响应来自设备的Ready To Transfer（RTT）UPIU 而创建的，而UPIU又由先前向Command UPIU发送到设备。 UTP engine使用来自相应的RTT UPIU和与原始命令UPIU相关联的PRD表的信息。过程如上图4所示。

#### 由Software解析的Inbound UPIUs

UTP engine 应分析任务标签，在适用的情况下分析UFS主机接收到的所有UPIU的LUN字段，以便它们可以匹配先前从UFS主机发送到UFS设备的UPIU（请求/响应匹配）具有task tag和LUN字段的UPIUs从UFS设备端传送过来被接收，并且匹配之前由UFS HOST发送的UPIU，它将被传送进入Host memory中相应的Descriptor。只有准备转移（RTT）和数据在UPIU中的处理方式不同，UTP Engine将进一步分析，以实现自主数据传输操作。对于host controller来说，接收到如Response，Task management Response等类型的UPIU是就标志着一个UTP Transaction的关闭。最终，UTRLDBR的相应位会被清除。

#### 由Host Controller进行解析的Inbound UPIUs

Data in 和 Ready to Transfer UPIU完全由Host controller/ UTP引擎处理，并且在处理它们时不涉及software。 数据在UPIU中携带从UFS设备检索的数据，并对其头信息进行解析，以允许Host controller 将包含的数据传输到host memory中的正确位置。RTT(Ready To Transfer) UPIU由主机控制器/ UTP引擎分析，以提取UFS设备提供的有关UTP引擎预期生成的下一个DATA OUT UPIU的必要信息。


----------


## UTP Transfer Request completion 的处理

当在 host controller 中接收到UTP传输请求完成(UTP Transfer Request Completion)时，处理的过程如下：

1. 如果完成是针对Regular Command（UTRD.I = 0）：
 - IA中断计数器递增。
 - 如果计数器在递增之前为0，则IA定时器开始运行
2. 当满足以下四个条件中的至少一个时，IS.UTRCS(UTP Transfer Request Completion Status)位置1：
 - UTRD.I 位被写‘1’（Interrupt Command）
 - 计数器在增长之后达到IACTH中的那个值（Interrupt aggregation counter threshold）
 - 当IA计时器达到在IATOVAL（Interrupt aggregation timeout value）中配置的值。（这种情况可能随时发生，不一定与请求完成相关）。
 -  完成命令的总体命令状态（OCS）不等于“SUCCESS”。

如果完成中断未被IE.UTRCE位屏蔽（禁用），则由写操作产生中断。host software 处理host controller 为command completion 产生的中断。 在中断服务程序中，host software 检查寄存器IS以确定是否有中断挂起。 如果设置了IS.UTRCS位，表示一个或多个UTP Transfer Requests（TR）已完成，应该采用以下步骤：
1. 如果IS寄存器中出现错误，则主机软件执行错误恢复操作。
2. 在IS.UTRCS中断的情况下，主机软件清除中断，然后可以使用两种方法之一来确定哪些UTP TRs已经完成：
 - 读取UTRLDBR寄存器，并将当前值与主机软件以前issue的仍然Outstanding的命令列表进行比较, 对于outstanding的任何一个TR，UTRLDBR中位置i（其中i表示issue Transfer Request的UTRL slot）中的值0意味着TR已经完成。 UTRLDBR是一个易失性寄存器; software 只能使用其值来确定已经完成的命令，而不能确定先前已经发出的命令。
 - 读取UTRLCNR寄存器。对于每一个TR, 其中bit i的值为1表示这个位置的TR已经完成。
3. 对于检测到完成的每个TR i，host software重复以下步骤：
 - 根据较高OS层（例如文件系统）的要求处理来处理 request completion.
 - 通过写'1'来清除UTRLCNR的相应位i。
 - 标记这个slot i表示可以用于重新使用了（仅限软件）
4. 在处理所有先前检测到的TR之后，软件可以通过向UTRIACR寄存器写入80010000h来重置和重新启动中断聚合机制。
5. 通过重复步骤2中描述的两种方法之一，软件确定步骤2之后是否已经完成了新的TR。 如果新的TR已经完成，则software重复步骤3中的序列。


----------


## Task Management Function

UTMRL(Task Management Request List)用于向附加的设备发送UTP任务管理功能。Host controller 应将Task management function放置在比UTP Transfer Request更高的优先级。当提交任务管理请求时，Host controller需要暂停所有active的UTP传输请求，转而分发task management 到相应的设备。

#### 创建一个 UTP Task Management Request 的基本步骤

当主机software需要向host controller发送UTP任务管理功能时，使用UTP任务管理请求列表(UTP task management request list)。与UTP transfer request类似，以下是主机软件构建UTP任务管理请求的步骤。

1. 通过读取UTMRLDBR找到一个空的transfer request slot。一个空的slot在UTMRLDBR中的相应位清零。
2. Host software 在empty slot出创建一个UTMRD.
3. 如果软件请求命令完成后设置HCS.UTMRCE，I（中断）位置1。
4. 用‘Fh’设置OCS（Overall Command Status）
5. 编程 Task Management Request UPIU.
6. 用‘0’来初始化字段 Task Management Response UPIU. 
7. 对每一个将要发送给host controller执行的task management function，重复step1 到 step 7
8. 检查UTMRLRSR寄存器并且保证在继续之前能够读取到'1'
9. 设置UTMRCE(UTP Task Management Request Completion Enable)使能位为‘1’来启动中断。
10. 设置UTMRLDBR来提醒doorbell register以向host controller表示多个transfer requests已经准备好被发送给相应的设备了。Host software只能在与新命令对应的bit position写‘1’。其他的UTMRLDBR应该写‘0’，表示对现有的值没有改变。

#### 处理UTP Task Management Completion

Host software 处理由host controller产生的命令完成的中断。在中断服务程序中，主机软件检查IS以确定是否有中断挂起。 如果UFSHCI有一个待处理的中断：
1. 主机软件通过读取IS寄存器来确定中断的原因。 如果设置了IS.UTMRCS位，则表示UTP任务管理请求已完成。
2. 主机软件根据中断原因清除IS寄存器中的相应位。
3. Host software 读取UTMRLDBR寄存器，并且将当前值与仍然还没有执行的之前由host software issue的命令列表进行对比。主机软件成功地结束相应位在相应寄存器中被清除的任何未完成的命令。 UTMRLDBR是一个易失性寄存器; 软件只能使用其值来确定已经完成的命令，而不是确定先前已经发出的命令。
4. 如果IS寄存器中出现错误，则主机软件执行错误恢复操作。

（完）

  [1]: http://blog.xxiong.me/usr/uploads/2018/12/3868930820.jpg
  [2]: http://blog.xxiong.me/usr/uploads/2018/12/3259455945.jpg
  [3]: http://blog.xxiong.me/usr/uploads/2018/12/2515594948.jpg
  [4]: http://blog.xxiong.me/usr/uploads/2018/12/744205718.png
