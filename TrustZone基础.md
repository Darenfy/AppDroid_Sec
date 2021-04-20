## 可信执行环境和 Arm 可信区
Trusted Execution Environments -> TEE
Arm TrustZone  ->  TZ

#### ARM 可信区技术
要保护应用程序

一方面内存破坏漏洞很普遍，另一方面把它们完全从大量的程序中消除是不现实的

所以对于一般程序来说，有一个解决办法就是把程序隔离到沙盒中，并强制所有交互都通过 "broker" 沙盒。
攻击者发现并利用程序中的漏洞就被限制性的与系统进行交互

而且 broker process 可以做的比其他程序小的多，加固它的代码就更可控

现代操作系统内核也很大，拥有相同的情况。
隔离内核比隔离普通程序复杂的多
device developers 可以利用 TEE
TEE 从主操作系统中隔离关键代码和数据，所以即使主操作系统被干掉，在TEE中的代码和数据依旧是被隔离的。
TEE的使用样例包括：验证操作系统自身的完整性，管理 User 证书（指纹传感器），存储和管理设备加密密钥。

#### A TZ 介绍
在操作系统被干掉的情况下保证数据安全需要特殊的硬件支持

跑在 Arm 上的设备可以使用 TZ 提供硬件层隔离来保证 TEE 安全。

Armv8-A配置文件提供TrustZone扩展，可用于具有集成V6或更高版本MMU的SoC。 
受TrustZone保护的代码和数据与恶意外围设备和非TrustZone代码隔离。 它可用于构建功能齐全的可信执行环境（TEE），包括在S-EL1上运行的TEE OS，与外围设备安全交互的可信驱动程序（TD）以及在S-EL0上运行的可信应用程序（TA） 

TrustZone还提供了一个安全监视器，该监视器可以在S-EL3的最高特权级别下运行，并且可以在所有模式下完全访问该设备。

TZ 通过引进CPU可以操作的新的“安全”模式运行
当操作系统运行在这个模式时，CPU 可以访问设备的所有硬件和内存
当运行在非TZ模式时，仅仅能访问部分外围设备和物理内存的特定范围

TEE 可以利用这个技术将他的代码和数据放在安全内存中，防止运行在普通模式下的代码访问或修改，即使代码运行在内核中。即使内核不能访问TEE内存，运行在TEE的代码可以读取修改和选择性处理普通模式的数据
![](https://azeria-labs.com/wp-content/uploads/2020/02/TZ-ELs-3.png)

TZ 同样扩展了CPU 的标准 Exeception Level 特权模型。

TZ 允许 CPU 运行在低权限下运行安全模式，允许TEE进行权限隔离

S-EL3   安全监视器运行设备制造商提供的 Arm Trusted Firmware（ATF）中的代码。 
该代码在 Rich Execution Environment（REE）和TEE内核之间执行上下文切换，并通过安全监视器调用（SMC）处理程序为这两者提供基本服务，REE和TEE OS可以通过SMC指令来请求这些服务。

S-EL1   取决于TEE的实现，TD 可能也跑在 S-EL1上。
没有等价的 S-EL2    因此，无法运行多个虚拟TEE操作系统
Armv8.4-A 配置文件里提出了 S-EL2 即 Arm Trusted Firmware

S-EL0   运行无特权的 TA 和 特定的 TEE 实现
没有被 TEE OS 明确允许的情况下 TA 不能操作其他 TA 的上下文，不能直接与外围设备通信，不能和 NWd 或者 SWd 的其他进程交互

#### Normal World vs Secure World
在 TZ 架构中，每一个逻辑处理器核心运行像有两个不同的虚拟核，一个在 TZ 内，一个在 TZ 外。

NWd内核像以前一样运行传统操作系统，并具有丰富的功能和正常的应用程序。
用TrustZone术语，整个环境都称为Rich Execution Environment（REE）。
相比之下，TrustZone虚拟核心在“安全世界”（SWd）中托管并运行受信任执行环境（TEE）。
实际上，TrustZone虚拟内核是通过在Secure Monitor中执行的快速上下文切换来实现的。

**SCR Register**
在硬件层，CPU 通过在 CP15 中 Secure Configuration Register（SCR）的 NS 比特位决定运行环境是 SWd 还是 NWd
bit -> 1 NWd  0 SWd
为了保证 SWd 的完整性，SM 确保这个比特位不会被非安全程序写

**通过系统总线的受保护外围设备访问**
CPU 可以标记内存页是属于 Secure 还是属于 Normal
Page Table Entry（PTE）的 NS bit（bit 19）确定...
该位确定通过系统总线访问基础物理内存时，是否将AxPROT[1]安全位设置为1，从而允许受TrustZone保护的操作系统和进程在同一地址空间中访问受TrustZone保护的安全内存和NWd内存。 
当处理器在TrustZone模式之外运行时，内存访问将始终忽略NS位，就像NS位被置位一样工作。

启用TrustZone的AMBA3 AXI总线结构中存在的硬件逻辑通过允许在总线传输期间将系统总线上的读写标记为“安全”或“非安全”，来保护SWD资源免遭NWd的访问。 
当安全总线主机启动总线传输操作时，ARPROT[1]或AWPROT[1]位确定是否应将传输视为安全交易还是不安全交易。 
此外，NS位在硬件中设置为高电平，从而防止非安全总线主设备访问安全总线从设备（安全主设备仍可以发出安全传输）。

#### 受 TZ 保护的内存
**TZASC**
为了严格的限制不受信任的代码和数据对受TZ保护的代码和数据的访问，TrustZone Access Space Controller TZ访问空间控制器硬件允许物理内存的特定区域被标记为“secure only”。
这些TZ区域通过TZASC寄存器设置-通过设备上的ATF配置。
改变访问表的起始或结束地址是特权操作，必须十分小心，错误的配置可能会导致不受信任的内存访问TrustZone内部使用的物理内存区域。

**页表**
在 NWd，SWd 内核和程序拥有他们自己的地址空间，通过页表管理
这些页表使MMU可以在CPU使用的虚拟地址之间快速转换为它们相应的物理页，并在访问内存时执行权限检查。

**虚拟MMU**
在TZ架构中，每一个CPU都有两个虚拟MMU 一个用来管理SWd地址空间 另一个管理NWd地址空间
每一个都有其相应的 TTBRx寄存器
这意味着两个模式的操作系统可以独立创建和管理虚拟页表集-他们和他们各自的进程所使用

**非安全内存**
TZ 页表可以将特定内存地址标记为非安全。
通过设置相关 PTE 的 NS 位实现
SWd代码可以使用它来将NWd内存的视图映射到其自己的（或TA的）地址空间，该地址空间可用于监视或与NWd通信。 
在NWd中，将忽略PTE中的NS标志。 来自NWd的所有内存访问均被视为不安全。

#### secure boot
使用TrustZone的设备还可以利用称为SecureBoot的技术从磁盘加载操作系统时增强操作系统的完整性。 这样可确保在关闭设备电源时，没有人篡改操作系统的代码。
 现代引导序列分为一系列阶段，每个阶段都会加载并验证下一个阶段的完整性。 
 安全启动提供了一种硬件信任根机制，该机制允许设备ROM验证第一阶段。 
 只要所有各个引导阶段都正确加载并检查下一个后续阶段的签名，并且只要数字签名密钥保持安全，此引导链就可以使用户确信他们的设备已引导未更改的副本 操作系统，即使他们无人看管设备也是如此。

对于 AFT 来说，bootloader序列如下：
- BL1 AP trusted ROM
- BL2 Trusted Boot Firmware
- BL3-1 EL3 Runtime Firmware
- BL3-2 Secure-EL1 Payload（optional）
- BL3-3 Non-trusted Firmware

#### 可信执行环境
![](https://azeria-labs.com/wp-content/uploads/2020/02/TEEs-minimal-overview.png)

TZ 是众多可以被用来构建可信执行环境的硬件特性的一种，旨在提供操作系统代码和数据的硬件和内存隔离

不是所有的 TEEs 都是使用 TZ。
Apple's Secure Enclave and Intel's SGX 

存在许多可以利用Arm TrustZone扩展的不同主要TEE：
- NVdia TLK
- Qualcomm QTEE
- OP-TEE
- Huawei iTrustee based on HiSilicon platform
- Trustonic Kinibi TEE-OS for TSP
- Samsung Teegris TEE
- ProvenCore TEE
- Trusty TEE for Android
- TrustKernel T6

**TEE的演变**
可信执行环境最初主要是作为一种提供安全启动功能的机制而开发的。
TEE逐渐发展为提供更多的功能并将服务提供给NWd。例如，支持多种功能，例如安全的指纹认证，加密服务，受保护的移动支付（如Samsung Pay，DRM，运行时完整性保护以及受保护的员工设备上的公司服务，例如通过以下方式在移动设备和公司基础设施之间提供安全的通信：TEE的套接字API。
现代TEE还可以通过“受信任的用户界面”保护设备上的敏感数据，该界面允许用户直接在TEE中输入PIN和密码，从而防止用户凭据对REE中运行的代码可见。此外，TEE还可以用于保护敏感数据，以防设备丢失或被盗。例如，设备可以选择将解密密钥专门保留在TEE内部以防止提取，并且可以设计设备解密系统，以便必须在TEE内部验证密码猜测并对其进行速率限制。 GlobalPlatform提出了在REE和TEE之间进行通信的API，并成为新的标准。

**TEE 组件**
TEE 依赖于硬件来实施强制隔离。
硬件组件提供基本体，允许安全代码的隔离和初始化。
TEE软件将在此硬件上运行，扩展这些原语，以提供功能齐全的执行环境，并支持各种功能，例如安全启动，安全监视器和轻量级TEE OS。 
TEE通常还会为REE应用程序提供支持，以将其自己的代码和数据的关键部分隔离到隔离的Trusted Application（TA）中。 第三方 TA 不直接属于TEE； 它们由TEE OS在TEE内部执行。 

它是硬件隔离，安全启动和受信任的TEE OS的组合，共同构成了功能齐全的TEE，可以在其上加载和运行TA。

**TEE 操作系统**
TEE操作系统是 NWd 操作系统的 SWd 的补充。 
它以比受信任的应用程序（TA）和受信任的驱动程序（TD）更高的特权级别运行。 
TEE OS支持与REE的通信，提供核心服务和对TA的访问，并为Trusted Drivers提供了环境。 
TEE OS处理从EL3发出的安全监视器调用（SMC），并处理来自在S-EL0中运行的TA的管理程序调用（SVC）请求。

#### 可信应用程序和可信驱动
**Normal TAs**
TA 运行在 S-EL0 层，使得 NWd 应用程序提供针对特定应用程序的功能，比如 DRM， 可信UI， 秘密和密钥的存储

TA由 TEE OS 加载并计划执行，并且与NWd应用程序，NWd内核以及其他TA隔离。 但是，给定的TA并非与TEE OS隔离开，必须隐含地信任设备上可用的TEE。 TEE OS的妥协必然会影响TEE内部运行的TA中数据和代码的完整性和机密性。 

最初，设备仅允许预先安装在设备上的TA。 但是，现代设备支持从TA应用程序商店下载的应用程序中包含的TA的动态部署。

**系统 TA**
系统TA是TA的子类，提供关键TEE OS服务，比如 安全存储，加密服务和TA认证与解密。
系统TA是通常都是由TEE OS 制造商或者OEM装载。
比如华为，又一个特权系统TA叫做 GlobalTask 实现一些可信OS的功能并控制normal TA的生命周期

**可信驱动** 
可信驱动程序包含访问安全外围设备（例如指纹读取器）或在TrustZone内部提供特权服务所需的代码。 
根据TEE，受信任的驱动程序要么在S-EL0中运行（例如Trustonic，Nvidia TEE），要么直接加载到TEE OS内核的地址空间中，并在S-EL1中运行（例如Qualcomm，华为，Linaro TEE）。 
受信任的驱动程序可以为TA提供附加服务，并且可以访问比TA更多的功能。

#### SWd NWd 通信
像移动支付应用程序之类使用TA的App 需要一种机制来构建 REE中NWd app 和 TEE中TA 的通信。

REE中的应用程序经过一个过程，以通过REE和TEE均可用的特定通信通道和API来加载TEE中的TA并与之通信。 
这些通信渠道扩大了攻击面并引入了新的攻击媒介。 
错误验证通过这些通信接口来自REE的输入会导致严重的特 权升级攻击。

**SMC-based Communication**
TEE 和 REE 代码通信一定会使用两种机制中的一种。
第一种机制使用SMC接口。 跑在 EL1 层的的REE OS （如果使用了 Hypervisor，则跑在EL2层）可以使用SMC指令从跑在S-EL3层的安全监控器请求系统服务，然后将其中继到 TEE OS

**特定硬件服务**
TEE 不仅提供服务支持EL0层应用程序装载以及和S-EL0层可信应用程序通信，通常也提供可以被直接EL1层NWd操作系统代码使用的特定硬件和特定TEE服务。
栗子：
TEE可能提供机制创建监控时钟，以便在设备变得无响应时重新启动设备，与板载SoC交互或在NWd和SWd之间共享内存。
在S-EL3和S-EL1中运行的安全代码提供了可以使用SMC指令请求的这些服务。该指令具有特权，必须从EL1调用，并由安全监视器的SMC调度处理程序处理。

**共享内存通信**
第二种通信的方式是使用共享内存，在NWd中运行的代码将一些物理内存映射到其地址空间，而TEE中的代码将相同的物理内存地址映射到其自己的地址空间，请注意：将内存标记为“非安全”。
然后，两个进程都可以看到从任一世界写入此共享内存缓冲区的内存。共享内存是一种有效的通信方式，因为它允许快速传输往返TrustZone的大量数据，而无需上下文切换。

## Trustonic‘s Kinibi TEE 实现
Trustonic’s kinibi TEE 实现主要为了使用猎户座芯片组的设备而设计，只要用在欧洲和亚洲市场

主要kinibi组件的概观
![](https://azeria-labs.com/wp-content/uploads/2020/03/Kinibi-overview-new-darkbg-2.png)

RTM:运行时管理器 负责处理内存管理，任务调度和异常处理，同样也可以在两个World中传递信息

MTK：Kinibi Microkernel 跑在 S-EL1层，提供任务间隔离，进程间通信和TA的抢式任务调度

Kinibi 客户端API：客户端应用程序使用 KMC API 装载并与TA 进程交互。
CA通过libMcClient.so库访问此API，该库提供API，以通过两个世界都可见的共享内存缓冲区来请求会话的句柄并在NWd和SWd之间中继通知。 
MobiCore客户端API的工作原理是将CA的请求中继到在NWd中以高权限运行的MobiCore守护程序。 该守护程序将请求中继到NWd EL1中的MobiCore驱动程序。 然后，该驱动程序通过MobiCore接口（MCI）与TEE RTM通信。

McLib：Mobicore库（McLib）由TEE中的TA和驱动程序使用，并且由专有的内部API和全局平台（GP）内部API组成。 受信任的应用程序可以使用TLAPI调用McLib函数，而受信任的驱动程序可以访问DRAPI提供的更丰富的存根集合。

加密操作和加密算法：Kinibi给REE提供了很多加密服务，包括安全随机数生成器，数据加密服务，安全密钥创建和销毁，同样提供了基础加密算法，比如加密哈希，MAC的生成与验证，数字签名

安全对象：Kinibi为REE应用程序提供了访问“安全对象”的能力，这些对象包含加密和完整性受保护形式的数据。加密后，数据可以存储在不安全的位置，例如NWd的磁盘上。 解密安全对象的密钥是从设备主密钥派生的，并且对于每个TA都是唯一派生的。 此外，安全对象的密钥永远不会离开TEE。 这意味着安全对象只能在生成安全对象的设备上，创建安全对象的TA内以及用户登录时解密。创建和管理安全对象的服务由Kinibi Trusted Storage API

#### 可信应用程序和可信驱动
![](https://azeria-labs.com/wp-content/uploads/2020/02/TEEs-minimal-Kinibi-blue.png)
**TA**
kinibi TA 和 TD 使用 Mobicore Loadable Format MCLF文件格式存储在磁盘中。
三星设备 TA 的 MCLF 文件通常存储在 /system/vendor/app/McRegistry /system/app/mcRegister /data/app/mcRegistry 文件夹 
![](https://azeria-labs.com/wp-content/uploads/2020/02/trustlets_location-1.png)

**TA APIs**
TA 运行在 S-EL0层。 TA通过McLib与TEE进行交互。这可以使得访问专有的内部API和全局平台（GP）内部API。 TA 通过tlApi接口调用McLib

**内嵌和系统TA**
TA被分为嵌入式TA和普通TA。Trustonic同样提供了一系列的系统TA，比如管理容器的内存管理TA，安卓门卫TA，由OEM提供的 keymaster system TA

**可信驱动**
在Kinibi中，可信驱动与TA拥有相同的权限执行，S-EL0.
但是，可信驱动可以通过 drApi 存根访问更加丰富的 McLib 功能集。这些附加的功能机允许 TD 访问额外的 Supervisor Calls（SVCs），映射物理内存，使用线程，调用SMC。TD 允许 TA 访问设备上的外围设备，这要求 TD 能够映射物理内存

**内嵌TD**
Kinibi捆绑了一个加密驱动程序（drcrypto）和安全存储服务驱动程序（STH2），两者均作为具有驱动程序特权的单独任务运行。 Drcrypto提供加密服务，而STH2为TA和TD提供安全的存储服务。 与TA不同，这些驱动程序直接嵌入在Kinibi中，而不是从磁盘加载或从REE加载。可以使用Quarkslab研究人员针对IDA Pro和Ghidra的T型装载机从引导装载程序sboot.bin中提取这些驱动程序。

#### 可信应用程序通信
apps in NWd 需要与 TAs in SWd 通信 
使用 Trusted Communication Interface TCL 的 Trustlet Application Application Connector

**Open Device**
CA 通过 mcOpenDevice 函数调用创建 device session。 
此函数将初始化与MobiCore设备的新连接以及与MobiCore实例进行通信所需的设备特定资源。 
该函数向 Kinibi Mobicore 守护程序发出请求。 反过来，这将打开虚拟设备 /dev/mobicore-user 的句柄
该虚拟设备用于与在NWd内核中运行的Mobicore驱动程序进行通信
![](https://azeria-labs.com/wp-content/uploads/2020/03/mcOpenDevice-672x1024.png)

**分配共享内存**
获得设备会话的句柄后，MWd CA 现在请求 Trustlet Connetor 通过 mcMallocWSM 函数创建 world-shared shared memory(WSM) 区。 Mobicore 驱动分配 WSM 区块，接下来使用这个区块作为 SWd TA 和 CA 的通信信道。在现在 Kinibi 实现中，共享缓冲区被映射到固定内存中 0x00100000 for TA and 0x00300000 for TD
![](https://azeria-labs.com/wp-content/uploads/2020/03/mcMallocWSM.png)

**Trustlet 会话**
NWd 中的 Trustlet Connector 接下来 尝试使用 mcOpenTrustlet 调用建立到TA的会话，提供设备的会话ID，指向包含要加载的TA的内存地址的指针以及Trustlet连接器接口（TCI）世界共享内存缓冲区的指针和长度。
TCI缓冲区的长度当前上限为1MiB（0x100000）。该mcOpenTrustlet请求被中继到NWd中的MobiCore驱动程序，然后由该驱动程序向SWd中的MobiCore发出请求。 然后，MobiCore通过MobiCore控制接口（MCI）代理创建新会话，并返回相应的会话ID。

#### TCL 缓冲区通信
建立会话后，将TA加载到TEE中，CA可以通过映射到两个地址空间的TCI缓冲区直接与其通信。
两者之间进行通信的确切协议是特定于应用程序的，但是典型的方法是使用TCI缓冲区的前4个字节发送命令ID，而缓冲区的其余部分保存相应命令的参数。 
将命令放入TCI缓冲区后，CA必须使用mcNotify函数通知TA该命令已准备好进行处理。此函数将通知中继到MobiCore驱动程序，后者再通过mc_notify通知Mobicore。
Mobicore接下来唤醒相应的会话端点，然后该端点可以处理传入的命令。NWd通过调用mcWaitNotification函数等待TA响应。 加载的TA可以使用tlApiWaitNotification调用来等待传入命令，该调用可通过TEE的McLib tlApi访问。这将暂停执行，直到CA调用mcNotify
![](https://azeria-labs.com/wp-content/uploads/2020/03/notify_TA-darkbg.png)

**TA 通知 CA**
被提醒输入命令后，TA读取TCL缓冲区查看应该运行哪一条 命令ID，并且使用TCI缓冲区中给定的参数处理相关命令。
完成请求后，TA将所有输出回写到TCI缓冲区中，接下里调用 tlApiNotify 函数。这被中继回NWd，从而导致mcWaitNotification调用完成。 然后，CA可以检查TCI缓冲区以查看其对TA的调用结果。
![](https://azeria-labs.com/wp-content/uploads/2020/03/notify_TLC-darkbg-1.png)

#### 使用 TD 通信
在 Trustonic TEE 设计中，TD 运行在 S-EL0，而不是 S-EL0.这意味着利用 TD 不会自动导致提权到 S-EL0。 然而，TD可以通过 McLib 提供的 drApi 与 MTK 通信。 这些 API 提供了强大的函数集，支持映射物理内存，创建线程，向安全监视器发起请求。
TD 可以通过 SVC 接口访问 TEE-OS 中的特权系统服务，比如映射物理内存的系统调用。
低特权TA 不能访问这些强大的功能，因此必须与特权 TD 通信来实现像与外围设备交互等行为
这个 TA-TD 通信信道不会暴露给 REE 
但是，在REE中的攻击者可能会破坏TA，通过此接口发送恶意数据以尝试控制TD，从而获得对通常仅可用于TD的特权API的访问权限。
因此，TA-TD通信信道也应被视为安全关键边界。

**硬编码白名单**
为了防止任意TA和任意TD通信，Trustonic使用带有允许与该驱动程序通信的TA UUID硬编码到相应TD中的白名单
🌰：ID为0x40002的Trustonic驱动程序仅将其攻击面限制为10个不同的TA。这意味着REE中的攻击者必须不仅危害任意TA，而且必须危害这些特定TA中的一个，才能与该TD进行交互

**攻击TD**
为了演示
为了演示攻击者如何通过CA加载易受攻击的TA并通过与TD通信来利用它来增加其攻击面，使用 ID 为0x1B000000 的 TA 内的内存损坏漏洞来调用 ID 0x40002（允许从该TA进行通信）的 TD。
TD 的 ID 可以从 TD 二进制标头信息中得出。

**TA 与 TD 交互**
当TA要与TD通信时，它需要调用 tlApi 的一部分 tlApi_CallDriver 或 tlApi_CallDriverEx 函数（取决于 TEE-OS 正在运行的 API 级别），并通过以下方式为该函数提供 TD ID 和参数寄存器
![](https://azeria-labs.com/wp-content/uploads/2020/03/callDriverApi_func2.png)

TD 接下来使用 drApiMapClientAndParams 调用通过 IPC handler 从 TA 地址空间到 TD 地址空间映射缓冲区。然后处理来自映射缓冲区的数据，以提取参数和命令，以供TD调用处理程序。
![](https://azeria-labs.com/wp-content/uploads/2020/03/drApiMapClientAndParams.png)

为了调用 TD，易受攻击的 TA 必须被加载和利用，以劫持 TA 内对程序计数器的控制。然后使用 ROP 链填充自变量寄存器。
一旦寄存器中填充了适当的参数，就可以通过分支到 tlApi 条目来调用 tlApi_callDriver 调用

Ref：
https://azeria-labs.com/trusted-execution-environments-tee-and-trustzone/
https://azeria-labs.com/trustonics-kinibi-tee-implementation/