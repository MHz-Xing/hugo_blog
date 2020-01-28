---
title : "认识BLE5协议栈[八]--链路层"
date : "2019-12-26T11:33:29+08:00"
tags : ["BLE5.0","协议栈","LL"]
series : ["BLE5协议栈"]
categories : ["无线BLE技术"]
draft : false
toc : true
---

链路层LL（Link Layer）是协议栈中最重要的一层。

链路层的核心是状态机，包含广播、扫描、发起和连接等几种状态，围绕这几种状态，BLE设备可以执行广播和连接等操作，链路层定义了在各种状态下的数据包格式、时序规范和接口协议。

<!--more-->

对于广播行为，链路层根据其可连接性，可扫描性，定向性三个维度定义了多种不同类型广播事件，相应的扫描行为和连接行为根据广播包的类型区分处理。连接过程涉及复杂的时序过程，利用连接参数可以配置连接过程时序。

广播、扫描和连接各自具有白名单过滤机制，可以针对指定地址的设备进行操作。链路层提供了一些列控制规程，比如加密连接和数据长度更新等，上层协议可以利用这些规程控制链路层。

此外，链路层利用私有地址实现了隐私特性。

## 1. 状态机
链路层设计了五种工作状态，涵盖了链路层的全部操作状态：

状态 | 描述
---|---
Standby | 统不做任何广播和扫描动作，可以维持低功耗。
Advertising | 系统对外发出广播数据和扫描响应数据。扫描响应数据也是一种广播数据，由扫描设备发出扫描请求，广播包设备返回扫描响应数据。
Scanning | 监听外部的广播数据。扫描状态并不能直接进入连接状态。
Initiating | 监听外部的广播数据。它可以发起连接请求，然后进入状态。
Connection |两个设备建立连接，进行通信。

这五种状态共同组成一个状态机，它们相互转换关系如下：

<center>![8-1：LL State Machine](/images/blog/BLE/8-1.png)</center>

观察上图，扫描态无法直接进入连接态，从待机状态进入连接状态通常发生在连接已经建立的情况。

如果设备从发起态进入连接态称为主角色（Master Role）或主设备，如果从广播态进入连接态称为从角色（Slave Role）或从设备。主设备发起连接请求，并且会设定连接过程的时序参数，从设备接受主设备设定的参数进行通信。

在一个时刻，状态机只能处于一种状态，而链路层可以同时拥有多个状态机。这就意味着BLE设备在一个状态机中保持连接状态的同时，另一个状态机保持广播状态，或者多个链路状态机同时处于连接状态，这是BLE设备实现多个连接的基础。

以下为同时建立多个连接的场景：

1. 如果设备A已经跟设备B保持连接，那么设备A可以执行广播或扫描操作。
1. 如果设备A已经跟设备B保持连接，并且设备A是主设备，那么设备A能够跟其他从设备C再次建立连接。
1. 如果设备A已经跟设备B保持连接，并且设备A是从设备，那么设备A能够跟其他主设备C再次建立连接。
1. 如果设备A已经跟设备B保持连接，那么不能实现设备A扫描，设备B广播并再次建立连接。设备A与B之间只能维持一个连接状态机。

## 2. 设备地址
设备地址代表了设备的唯一识别码，它是设备间相互识别的依据，不同的设备必须具有不同的设备地址。

设备地址分为多种类型，最简单的是Public Address。这种地址固定不变，可以根据设备地址跟踪到该设备。

Public Address属于一种48-bit的MA-L类型地址，它的结构为：NN:NN:NN:NN:NN:NN。

其中前三个字节使用OUI（Organizationally Unique Identifier），后三字节自由分配。OUI代表了一个指定的组织机构识别码，全球已经有许多科技公司申请了自己的OUI。开发者可以从IEEE网站上下载已被分配的MA-L地址，也可以从第三方网站查询某个公司的OUI或某个OUI对应的公司。

与Public Address不同，Random Address采用一个随机数作为地址，Random Address分为Static Address和Private Address两类，Private Address又分为Resolvable private address和Non-resolvable private address两类。

它们的特点如下表所示：

地址类型 | 是否可变 | 更新触发机制 | 作为识别地址 | 描述
---|---|---|---|---
1. Public Address | 固定 | 无 | Yes | 固定不变，需要避免地址冲突。
2. [Random Address] |  | | 		
2.1 Static Address | 随机 | 设备重新上电 | Yes | 设备上电时产生新的随机地址。
2.2 [Private Address] | | | 				
2.2.1 Resolvable private address | 随机 | 定时 | No | 过一段时间更新随机地址，链路层推荐更新周期为15分钟。该地址可以通过地址识别密钥解析识别。
2.2.2 Non-resolvable private address | 随机 | 定时 | No | 	过一段时间更新随机地址，链路层推荐更新周期为15分钟。该地址可以不可被解析识别。

一个设备可以具有多种类型的地址，但是必须拥有一个识别地址。Public Address和Static Address可作为识别地址。所以如果设备采用Resolvable private address地址类型，必须同时具有一个Public Address或Static Address地址。

## 3. 物理信道
在物理层的介绍中，提到了BLE将2.4GHz频段分成了40个物理信道，相邻信道频率间隔为2MHz。

这40个物理信道分成以下几类：

信道类型 | 信道号 | 描述
---|---|---
Primary Advertising Channel | 37、38、39 | 发送传统广播和扫描请求
Secondary Advertising Channel | 0 – 36 | 发送扩展广播
Periodic Advertising Channel | 0 – 36 | 发送周期广播
Data Channel | 0 – 36 | 发送连接数据

## 4. 数据包结构
### 4.1 非编码型物理层
非编码型物理层对应的链路层数据包结构包含四个部分：前导码，访问地址，PDU和CRC。如下所示：

字段 | Preamble | Access Address | PDU | CRC
---|---|---|---|---
长度 |1 or 2 octets	| 4 octets | 2 – 257 octets | 3 octets
传输时间(1M PHY/2M PHY) | 8 us/8 us | 32 us/16 us | 16 – 2056 us/8 – 1028 us | 24 us/12 us

- 前导码：前导码内容为重复的0/1或1/0序列，用于频率同步、符号时序计算和自动增益控制。对于1M速率的物理层，其长度为1字节；对于2M速率的物理层，其长度为2字节。
- 访问地址：用于区分不同的数据类型。对于主要广播信道的数据包，其访问地址是一个固定值，对于其他信道的数据包，其访问地址是一个随机值。
- PDU：包含了有效数据。
- CRC：针对PDU部分进行校验。假如PDU经过加密，则校验加密后的PDU。

### 4.2 编码型物理层

编码型物理层对应的链路层数据包做了FEC编码处理，增加了几个字段，其结构如下：

字段 | Preamble | Access Address | CI | TERM1 | PDU | CRC | TERM2
---|---|---|---|---|---|---|---
长度 | 10 octets | 4 octets | 2 bits |3 bits | 2 – 257 octets | 3 octets | 3 bits
传输时间(1M Coded PHY) | 80 us | 256 us | 16 us | 24 us |N×8×S us | 24×S us | 3×S us

其中Access Address、CI和TERM1属于FEC block 1，PDU、CRC和TERM2属于FEC block 2。FEC block 1采用S=8编码算法，FEC block 2根据CI字段值采用S=2或S=8编码算法。

编码型物理层的符号传输速率是1M Sym/s，当采用S=2编码，数据传输速率为500bit/s，当采用S=8编码，数据传输速率为125bit/s。

- 前导码：前导码内容为重复的00111100序列，它不执行FEC编码。
- 访问地址：与上相同。
- CI：编码指示器，如果CI=0，则FE block 2执行S=8编码，如果CI=1，则FEC block 2执行S=2编码。
- TERM1和TERM2：FEC算法终止符。

## 5 广播信道PDU
广播信道包括主要广播信道和次要广播信道，在这些信道中可以传输广播数据、扫描数据、发起连接数据和扩展广播数据。

广播信道中的传输的数据PDU结构如下：

字段 | Header | Payload
---|---|---
长度 | 2 octets | 1 – 255 octets

其中Header字段的结构如下：

字段 | 	PDU Type | RFU | ChSel | TxAdd | RxAdd | Length
---|---|---|---|---|---|---
长度 | 4 bits | 1 bit | 1 bit | 1 bit | 1 bit | 8 bits

- PDU Type：PDU数据的类型，具体而言就是不同的广播数据。
- RFU：For Future Use，即暂时不用。
- ChSel：信道选择。
- TxAdd：发送数据的设备地址类型，如果该位是0表示public address，1表示random address。
- RxAdd：接收数据的设备地址类型，如果该位是0表示public address，1表示random address。
- Length：Payload的长度。有效范围为1 – 255字节。

PDU Type的可选值如下表：

PDU Type bits | PDU类型 | 分类
---|---|---
0000b | ADV_IND | 广播PDU
0001b | ADV_DIRECT_IND | 广播PDU
0010b | ADV_NONCONN_IND | 广播PDU
0011b | SCAN_REQ | 扫描PDU
0011b | AUX_SCAN_REQ | 扫描PDU
0100b | SCAN_RSP | 扫描PDU
0101b | CONNECT_IND | 连接PDU
0101b | AUX_CONNECT_REQ | 连接PDU
0110b | ADV_SCAN_IND | 广播PDU
0111b | ADV_EXT_IND | 广播PDU，扩展广播PDU
0111b | AUX_ADV_IND | 广播PDU，扩展广播PDU
0111b | AUX_SCAN_RSP | 扫描PDU，扩展广播PDU
0111b | AUX_SYNC_IND | 广播PDU，扩展广播PDU
0111b | AUX_CHAIN_IND | 广播PDU，扩展广播PDU
1000b | AUX_CONNECT_RSP | 	连接PDU，扩展广播PDU
### 5.1 广播PDU
PDU类型 | Payload内容 | 适用的广播事件
---|---|---
ADV_IND | 广播设备地址，广播数据 | connectable and scannable undirected advertising event
ADV_DIRECT_IND | 广播设备地址，目标设备地址 | connectable directed advertising event
ADV_NONCONN_IND | 广播设备地址，广播数据 | non-connectable and non-scannable undirected advertising event
ADV_SCAN_IND | 广播设备地址，广播数据 | scannable undirected advertising event
ADV_EXT_IND | 广播设备地址，目标设备地址，ADI，Aux指针，发送功率等级 | 除了connectable and scannable undirected advertising event之外全部事件
AUX_ADV_IND | 广播设备地址，目标设备地址，ADI，Aux指针，同步信息，发送功率等级，ACAD，广播数据 | 除了connectable and scannable undirected advertising event之外全部事件
AUX_SYNC_IND | Aux指针，发送功率等级，ACAD，广播数据 | periodic advertising event
AUX_CHAIN_IND | ADI，Aux指针，发送功率等级，广播数据 | 除了connectable and scannable undirected advertising event之外全部事件

其中，AUX缩写表示辅助（Auxiliary），EXT缩写表示扩展（Extend）。

### 5.2 扫描PDU
PDU类型 | Payload内容
---|---
SCAN_REQ | 扫描设备地址，广播设备地址
SCAN_RSP | 广播设备地址，扫描响应数据
AUX_SCAN_REQ | 扫描设备地址，广播设备地址
AUX_SCAN_RSP |广播设备地址，Aux指针，发送功率等级，ACAD，广播数据
### 5.3 连接PDU
PDU类型 | Payload内容
---|---
CONNECT_IND | 发起设备地址，广播设备地址，LLData
AUX_CONNECT_REQ | 发起设备地址，广播设备地址，LLData
AUX_CONNECT_RSP | 广播设备地址，目标设备地址

其中LLData包含了建立连接需要用到的参数信息，如下：

字段 | AA | CRCInit | WinSize | WinOffset | Interval | Latency | Timeout | ChM | Hop | SCA
---|---|---|---|---|---|---|---|---|---|---
长度 | 4 octets | 3 octets | 1 octet | 2 octets | 2 octets |2 octets |2 octets | 5 octets | 5 bits | 3 bits

- AA：链路层的访问地址。
- CRCInit：校验运算的初值，由链路层随机生成。
- WinSize：建立连接时的连接窗口。
- WinOffset：建立连接时的连接窗口偏移量。
- Interval：连接间隔。
- Latency：从设备的握手潜伏期。
- Timeout：从设备断开的超时时间。
- ChM：信道占用图。该参数共40bit，第1位表示信道0，第2位表示信道1，以此类推。前37个比特位代表37个数据信道，如果对应的信道曾经被使用过，则设置1，否则设置0。
- Hop：跳频算法的增量，它的范围是5-16。
- SCA：睡眠时钟精度（Sleep Clock Accuracy）。

SCA的取值与睡眠时钟精度的关系如下表：
SCA | 睡眠时钟精度
---|---
0 | 250ppm – 500ppm
1 | 150ppm – 250ppm
2 | 100ppm – 150ppm
3 | 75ppm – 100ppm
4 | 50ppm – 75ppm
5 | 30ppm – 50ppm
6 | 20ppm – 30ppm
7 | 0 – 20ppm

### 5.4 扩展广播PDU
扩展广播是BLE 5新增的广播类型，这里将他们单独提取出来如下：

- ADV_EXT_IND
- AUX_ADV_IND
- AUX_SCAN_RSP
- AUX_SYNC_IND
- AUX_CHAIN_IND
- AUX_CONNECT_RSP

这些PDU具有相同的数据结构：

字段 | Extended Header Length | AdvMode | Extended Header | AdvData
---|---|---|---|---
长度 | 	6 bits | 2 bits | 0 – 63 octets | 0 – 254 octets

- Extended Header Length：Extended Header字段的长度，它的有效范围是0到63。
- AdvMode：广播事件发生时该设备所处的模式，它与各种模式的对应关系如下：

AdvMode | 模式
---|---
00b | Non-connectable + non-scannable
01b | Connectable + non-scannable
10b | Non-connectable + scannable

- Extended Header：扩展协议头，其结构如下：

字段 | Extended Header Flags | AdvA | TargetA | RFU	AdvData | Info(ADI) | AuxPtr | yncInfo | TxPower | ACAD
---|---|---|---|---|---|---|---|---|---
长度 | 1 octet | 6 octets | 6 octets | 1 octet | 2 octets | 3 octets | 18 octets | 1 octets | varies

各字段含义：
- Extended Header Flags：扩展协议头各个字段的开关标志位。该字段有8个比特位，前7位分别对应AdvA、TargetA、RFU、ADI、AuxPtr、SyncInfo、TxPower，如果置1表示包含该字段，如果置0表示不包含。第8个比特位不与ACAD字段关联，暂不使用。
- AdvA：广播设备地址。
- TargetA：扫描设备或发起设备地址。
- ADI：广播数据信息。该字段分两部分：DID（Advertising Data ID）和SID（Advertising Set ID），如果一个广播包由多个子包组成，那么这些子包的SID相同，DID不同，如果DID也相同，则说明该包内容与前包内容一样。
- AuxPtr：辅助广播包的指针（Auxiliary Pointer）。它告诉链路层下一个辅助广播包的信道号和时间偏移。
- SyncInfo：周期广播包信息。‘
- TxPower：设备输出功率。
- ACAD：额外的广播数据（Additional Controller Advertising Data）。它能包含少量的广播数据，可以配合普通广播包使用。
- AdvData：扩展广播数据，单个PDU可达255字节。

## 6. 数据信道PDU
数据信道PDU结构如下：

字段 | Header | Payload | MIC(Optional)
---|---|---|---
长度 |2 octets | 0 – 251 octets | 4 octets

Header：协议头。其结构如下：

字段 | LLID | NESN | SN | MD | RFU | Length
---|---|---|---|---|---|---
长度 | 2 bits | 1 bit | 1 bit | 1 bit | 3 bits | 8 bits

其字段含义如下：
- LLID：链路ID（Link Layer ID），用以区分该PDU是一个数据还是控制命令。当LLID=11b，表示该PDU是一个控制命令。
- NESN：下一个期望的PDU序号（Next Expected Sequence Number）。
- SN：当前PDU序号（Sequence Number）。SN与NESN合作，可以推测出该PDU是一个新数据，还是对上一个老数据的重传，从而实现流程控制。
- MD：数据未完（More Data），表示后续还有数据片段。
- Length：Payload+MIC的总长度，有效范围为0-255字节。
- Payload：装载L2CAP层数据包或链路层控制命令。对于数据包，最大Payload长度为251字节。L2CAP数据可能被分段，如果当前Payload是L2CAP数据包的开头或结尾片段，则LLID=10b，否则LLID=01b。
- MIC：完整性检测（Message Integrity Check）。用于确认加密的Payload是否完整有效。当Payload不加密或者Payload长度为0时，MIC字段不存在。

如果传输链路层可控制命令，则将控制命令装载在Payload字段中。

控制命令的结构如下：

 NULL| Opcode | CtrData
 ---|---|---
长度 | 1 octet | 0 – 26 octets

- Opcode：操作码，代表不同的控制命令。
- CtrData：控制参数。

链路层的所有控制命令如下：

Opcode | Control PDU Name | Opcode | Control PDU Name
---|---|---|---
0x00 | LL_CONNECTION_UDPATE_IND | 0x0D | LL_REJECT_IND
0x01 | LL_CHANNEL_MAP_IND | 0x0E | LL_SLAVE_FEATURE_REQ
0x02 | LL_TERMINATE_IND | 0x0F | LL_CONNECTION_PARAM_REQ
0x03 | LL_ENC_REQ | 0x10 | LL_CONNECTION_PARAM_RSP
0x04 | LL_ENC_RSP | 0x11 | LL_REJECT_EXT_IND
0x05 | LL_START_ENC_REQ | 0x12 | LL_PING_REQ
0x06 | LL_START_ENC_RSP | 0x13 | LL_PING_RSP
0x07 | LL_UNKNOWN_RSP | 0x14 | LL_LENGTH_REQ
0x08 | LL_FEATURE_REQ | 0x15 | LL_LENGTH_RSP
0x09 | LL_FEATURE_RSP | 0x16 | LL_PHY_REQ
0x0A | LL_PAUSE_ENC_REQ | 0x17 | LL_PHY_RSP
0x0B | LL_PAUSE_ENC_RSP | 0x18 | LL_PHY_UPDATE_IND
0x0C | LL_VERSION_IND | 0x19 | LL_MIN_USED_CHANNELS_IND

上述26个控制命令，可以分为以下几类：

- 加密：包括加密开始、暂停。
- 连接控制：包括连接参数更新，连接终止，PING。
- 信息交换：包括链路层功能交换，物理层信息交换，版本信息，信道图信息。
- 扩展功能：物理层更新，包长度更新。
- 杂项：错误响应、命令拒绝。
## 7. 数据流
### 7.1 非编码型物理层

对于非编码型物理层，数据处理流程如下：

<center>![8-2：Process_Uncoded_PHY](/images/blog/BLE/8-2.png)</center>

上面一行表示发送过程，数据流从左至右，BLE数据先进行加密，然后生成CRC校验信息，再进行白化（Whiten），从天线发射出去。

下面一行表示接收过程，数据流从右至左，执行发送过程的逆过程。

白化过程对数据序列执行多项式变化，使连续的0或连续的1数字序列被打散。因为接收机长时间接收0或1会误以为信号发生了频偏。

反白化则将白化数据还原成原始数据。由于白化和反白化是公开可逆的，所以它们无需加密。

BLE数据接收时，再进入反白化之前，首先检查访问地址是否正确，检测失败的数据会被抛弃。CRC校验失败的数据也会被抛弃。

### 7.2 编码型物理层
相对于非编码型物理层，编码型物理层多了编解码过程，数据的处理流程如下：

<center>![8-3：Process_Coded_PHY](/images/blog/BLE/8-3.png)</center>

编码过程包含前向纠错编码（FEC）和模式映射（Pattern Mapper）两个子过程。

前向编码将原始数据做卷积处理，处理后1个比特原始数据变成2个比特。

模式映射对卷积结果进行展宽，展宽映射如下：

卷积结果 | 模式映射（S=2） | 模式映射（S=8）
---|---|---
0 | 0 | 0011
1 | 1 | 1100

最终，S=2情况下1个比特变成了2个比特，S=8情况下1个比特变为8个比特。冗余的比特可以用来自矫正，从而减少重传次数，间接的提升了接收灵敏度。

## 8. 空中接口协议
空中接口协议定义了链路层在各种状态下的行为和时序规范。

### 8.1 帧间隔
**帧内间隔T_IFS（Inner Frame Space）**

帧内间隔表示前一帧的末尾与下一帧的开头之间的时间。

这个间隔时间为固定150us，通常表示为T_IFS = 150us。

**最小辅助帧间隔T_MAFS（Min AUX Frame Space）**

假如一个帧包含AuxPtr，最小辅助帧间隔表示该帧末尾与其辅助帧开头之间的时间。

这个间隔时间为固定300us，通常表示为T_MAFS = 300us。

### 8.2 时钟精度
链路层需要用到两个时钟精度参数，广播事件和连接事件使用活动时钟精度，其他事件则使用睡眠时钟精度。

活动时钟精度为±50ppm，睡眠时钟精度为±500ppm。

活动时钟驱动的行为事件，其时序误差应小于±2us，睡眠时钟驱动的行为事件，其时序误差应小于16us。

除此之外，BLE 5引入了一个距离延迟（Range Delay）的概念。

由于BLE 5极大的扩展了通信距离，假如两个设备相距1km，那么电磁波在空间传输，会产生一个时延，称为距离延迟。

电磁波速度为光速，考虑通信介质（空气），做了一个保守估计1/c = 4ns。

于是可以得到：距离延迟T(range) ≈ 2Distance * 4ns。

（蛋疼）

### 8.3 设备过滤机制
链路层基于设备地址执行设备过滤机制，利用一个白名单，记录设备的地址和地址类型。

广播状态、扫描状态和发起状态三种状态下的过滤机制相互独立，拥有各自的过滤策略，但是三种过滤机制共享一个白名单。

过滤机制 | 	描述
---|---
广播过滤机制 | 根据白名单过滤扫描和连接请求
扫描过滤机制 | 根据白名单过滤广播数据
连接过滤机制 | 根据白名单连接广播设备
### 8.4 广播事件
在广播状态下，链路层在广播事件中发送广播数据PDU，一个广播事件中可以发送多个广播数据PDU。

BLE 5扩展了广播能力，可以按照新增的广播功能对广播事件进行分类，如下：

广播事件 | 广播信道 | 描述
---|---|---
Advertising Event | 主要广播信道 | 传统的广播事件
Extended Advertising Event | 次要广播信道 | 扩展的广播事件，使用带AUX_ADV_IND
Periodic Advertising Event | 次要广播信道 | 周期的广播事件，使用AUX_SYNC_IND

**传统的广播事件**

广播事件是指完成一次广播数据的发送。

一次广播事件中，设备依次在37、38、39三个信道上传输相同的广播数据，并监听扫描请求和连接请求。收到请求时如果扫描过滤策略或连接策略允许，则做相应的响应，否则关闭本次广播事件或跳到下一个主要广播信道继续广播。如下图所示：

<center>![8-4：Scan_Req_and_Rsp](/images/blog/BLE/8-4.png)</center>

两个相邻的广播事件之间的时间差称为广播间隔。广播间隔是一个整数乘以0.625ms，有效范围是20ms至10485.759375s。

两个相邻广播之间会加入一个0-10ms的随机时延，称为advDelay。下图为三次广播事件，依次发生在37、38、39广播信道上：

<center>![8-5：ADV_Interval](/images/blog/BLE/8-5.png)</center>

实际上两次广播事件之间的时间略大于广播间隔，T_advEvent = advInterval + advDelay。

**扩展的广播事件**

扩展的广播事件可以通过多个辅助PDU来传输广播数据，极大扩展了广播数据的容量。

扩展广播事件包含了一个传统的广播PDU，以及一些列辅助广播PDU，如下图所示：

<center>![8-6：Extended_Adv_Event](/images/blog/BLE/8-6.png)</center>

**周期的广播事件**

周期广播，以一个恒定的连接间隔进行广播，广播一旦开始就不能更改广播间隔。

周期广播，使用AUX_SYNC_IND作为广播周期的标识，两个相邻的AUX_SYNC_IND PDU之间的事件间隔称为周期广播事件的广播间隔。如下所示：

<center>![8-7：Periodic_Adv_Event](/images/blog/BLE/8-7.png)</center>

广播间隔是一个整数乘以1.25ms，有效范围是7.5ms至81.91875s。

**广播事件类型**

按照能否连接、能否接受主动扫描和是否定向三个维度可以将广播事件做如下分类：

广播事件 | 可连接/可扫描/定向 | 可用PDU类型 | 广播信道
---|---|---|---
connectable and scannable undirected event | Y/Y/N | ADV_IND | 主要广播信道
connectable undirected event | Y/N/N | ADV_EXT_IND | 次要广播信道
connectable directed event | Y/N/Y | ADV_DIRECT_IND, ADV_EXT_IND | 主要广播信道，次要广播信道
non-connectable and non-scannable undirected event | N/N/N | ADV_NONCONN_IND, ADV_EXT_IND | 主要广播信道，次要广播信道
non-connectable and non-scannable directed event | N/N/Y | ADV_EXT_IND | 次要广播信道
scannable undirected event | N/Y/N | ADV_SCAN_IND, ADV_EXT_IND | 主要广播信道，次要广播信道
scannable directed event | N/Y/Y | ADV_EXT_IND | 次要广播信道

### 8.5 扫描事件
链路层在扫描状态下在主要广播信道监听广播数据。被动扫描不发任何数据，主动扫描发出扫描请求并监听扫描响应。

扫描行为的持续过程称为扫描窗口scanWindow，两次扫描行为之间的时间间隔成为扫描间隔scanInterval，显然扫描窗口不能大于扫描间隔，如果扫描窗口等于扫描间隔，链路层将持续扫描。

扫描间隔的最大值是40.96s。

假如广播设备扩展广播，链路层还需监听次要广播信道中的辅助广播包。

链路层监听到广播数据，或接收到扫描响应数据，则向主机发出广播报告（Advertising Report）。

### 8.6 连接事件
链路层属于Master Role的设备称为主机（master），属于Slave Role的设备称为从机（slave）。

一旦进入连接状态，就视为建立了连接。建立连接后，主机会发出一个数据并等待从机的响应，如果在6个连接间隔内都未等到从机的数据包，则视为连接断开。

在连接状态下，两端设备发送数据包的最小单元称为连接事件。链路层仅在连接事件中发送PDU数据，在一个连接事件内，可以发出多个PDU，相邻PDU之间至少保留T_IFS时间。

### 8.6.1 连接参数
连接事件的时序受两个参数的影响：连接间隔和从机握手潜伏数。

两个连接事件之间的事件间隔称为连接间隔。连接间隔是一个整数乘以1.25ms，有效范围是7.5ms至4s。

从机无需监听每一次主机的连接事件，忽略的事件总数称为从机握手潜伏数。主机设置一个监听超时，当主机等待超时仍未获得从机的响应，则认为连接断开，并向主机报告。监听超时是一个整数乘以10ms，有效范围为100ms至32s。

### 8.6.2 连接过程 – 主机端
主机在发出CONNECT_REQ之后，即进入连接状态，然后等待一会时间，发送第一个数据包。

等待的时间主要是为了留下空余让从机有充分时间唤醒和准备。

等待的时间包括三个参数：transmitWindowDelay，transmitWindowOffset和transmitWindowSize。其中第一个参数对于不同的物理层实现是一个固定值，后两项则可以通过主机进行设置。

主机连接过程的时序图如下：

<center>![8-8：Connection_Setup_Master](/images/blog/BLE/8-8.png)</center>

主机的一个数据包总是在发送窗口（Transmit Window）中发送，发送窗口的时序位置由以上三个参数共同确定。

辅助广播包引起的连接过程，与上图基本一致。

### 8.6.3 连接过程 – 从机端
在建立连接时，从机端需要监听主机端发出的第一包数据。假如从机错过了传输窗口，则在下一个连接间隔中监听主机第一包数据。如下图所示：

<center>![8-9：Connection_Setup_Slave](/images/blog/BLE/8-9.png)</center>

辅助广播包产生的连接过程，与上图基本一致。

### 8.6.4 关闭连接事件
关闭连接事件并非断开连接，仅表示当次数据传输事件完毕。

当连接事件中仅有一个PDU需要发送，则发送完毕后即可关闭连接事件。

如果连接事件中有多个PDU需要发送，那么将在PDU中设置MD（More Data）字段，在一个PDU发送完毕后持续发送，直到全部PDU发送完毕，再关闭连接事件。

每次主机发送一个PDU，从机都需要返回一个响应。假如从机没有返回响应，则主机中断发送，关闭连接事件。如果PDU没有发送完毕，从机没有收到主机发送的数据包，则从机端关闭连接事件。如果PDU的CRC校验失败，则关闭连接事件。

### 8.6.5 窗口展宽
连接事件的时序由睡眠时钟决定，而两端设备的睡眠时钟的精度均是±500ppm，因此从机为了收到主机的PDU，需要提前一段时间唤醒以监听主机的数据。

这段提前唤醒的监听时间，称为“窗口展宽”。显然提高睡眠精度将减少从机的窗口展宽时间。

windowWidening = (( masterSCA + slaveSCA ) / 1000000) * timeSinceLastAnchor

其中masterSCA表示主机的睡眠时钟精度，slaveSCA表示从机的睡眠时钟精度，timeSinceLastAnchor表示上一次

### 8.7 流程控制
数据信道PDU中有两个参数可以实现确认机制：SN，NESN。其中SN表示当前数据包的序号SeqNum，NESN表示下一个期望包的序号NextExpectedSeqNum。

SN和NESN字段长度均为1比特，在连接事件开始时，二者均设置为0，之后在0与1之间变换。

设备接收数据包时，如果发现SN和NESN相同，表明该数据包为新数据，则跳变NESN。如果发现SN和NESN不同，表明上次发送从机未收到，该数据位重发数据，则不跳变NESN。

设备发送数据包时，如果发现SN和NESN不同，表明上次数据被成功收到，则发送新的数据并跳变SN。如果发现SN和NESN相同，表明上次数据未被成功收到，则重发上次数据且不跳变SN。

归纳数据接收和发送两种情况，如下图所示：

<center>![8-10：LL_Ack_Scheme](/images/blog/BLE/8-10.png)</center>

数据信道PDU中还包括MD（More Data）字段，表示当前数据包后面是否还有更多数据包。

利用SN、NESN、MD三个字段，即可实现长包发送的流程控制。

### 8.8 PDU长度
链路层的连接状态PDU长度范围如下：

是否支持数据包扩展特性 | 是否支持编码性物理层 | 最大长度（字节） | 最大传输时间（微秒）
---|---|---|---
N | N | 27 | 328
Y | N | 27-251 | 328-2120
N | Y | 27 | 328-2704
Y | Y | 27-251 | 328-17040

根据上表，对于不支持扩展数据包和非编码型物理层的设备，链路层传输一个27字节PDU的时间为328us。

### 8.9 特性支持

设备可以选择性支持链路层的功能特性，对于两端设备均支持的功能特性，才可以使用。

链路层的功能特性列表如下，许多特性与链路层控制命令一致。

# | 特性
---|---
0 |	LE Encryption
1 |	Connection Parameters Request Procedure
2 |	Extended Reject Indication
3 |	Slave-initiated Features Exchange
4 |	LE Ping
5 |	LE Data Packet Length Extension
6 |	LL Privacy
7 |	Extended Scanner Filter Policies
8 |	LE 2M PHY
9 |	Stable Modulation Index – Transmitter
10 |	Stable Modulation Index – Receiver
11 |	LE Coded PHY
12 |	LE Extended Advertising
13 |	LE Periodic Advertising
14 |	Channel Selection Algorithm #2
15 |	LE Power Class 1
16 |	Minimum Number of Used Channels Procedure

## 9. 链路层控制
链路层定义了控制链路的规程，利用这些规程，更新链路层的参数，如下表：

规程 | 描述
---|---
Connection Update Procedure | 更新连接参数，仅在“连接参数请求”被拒绝的情况下，才执行该规程
Channel Map Update Procedure | 更新信道映射图，参与跳频信道选择
Encryption Procedure | 加密数据包
Feature Exchange Procedure | 交换所支持的特性
Version Exchange | 交换版本信息，包括链路层版本，公司ID等
Termination Procedure | 断开连接规程
Connection Parameters Request Procedure	| 更新连接参数请求规程
LE Ping Procedure | Ping规程
Data Length Update Procedure | 更新最大PDU长度规程
PHY Update Procedure | 物理层更新规程
Minimum Number Of Used Channels Procedure | 使用最少信道规程

链路层应该单线程操作规程，并且设置超时，如果同时进行两个不兼容的规程，将导致断开连接。

## 10. 隐私

链路层的隐私特性就是使用可解析的私有地址。

链路层维护一个计时器，当计时器触发，则更新私有地址，如果连接断开，也更新私有地址。协议栈推荐该计时器的周期为15分钟。

如果广播设备使用了可解析的私有地址，则PDU中的广播设备地址字段使用本地的IRK生成。

如果发起设备使用了可解析的私有地址，则PDU中的设备地址字段可以使用该私有地址。

如果广播设备收到了使用了可解析私有地址的扫描设备或发起设备的请求，则需要解析该地址。

如果发起设备收到了使用了可解析私有地址的广播设备的广播数据，则需要解析该地址。
