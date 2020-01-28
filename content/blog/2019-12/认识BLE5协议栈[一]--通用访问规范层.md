---
title : "认识BLE5协议栈[一]--通用访问规范层"
date : "2019-12-26T10:36:45+08:00"
tags : ["BLE5.0","协议栈","GAP"]
series : ["BLE5协议栈"]
categories : ["无线BLE技术"]
draft : false
toc : true
---

通用访问规范GAP（Generic Access Profile）是BLE设备内部功能对外的接口层，它规定了三个方面：GAP角色、模式和规程、安全问题。
<!--more-->

GAP层将设备分为四种角色，分别是外围设备，中央设备，播报设备和观察设备。这些设备围绕着广播和连接的差异性而区分，外围设备和播报设备对外发出广播数据，中央设备和观察设备扫描外部广播数据，播报设备和观察设备通常不建立连接，而外围设备和中央设备可以建立连接。

围绕着广播和连接，GAP层定义了许多不同的广播模式和连接模式，在不同模式下的操作称为“规程”。

对于安全问题，GAP层提供了BLE安全管理器的一些列参数设置接口，包括：设备的安全模式、IO能力、安全级别和密钥长度等参数。设备需要根据实际能力设置GAP的安全管理器参数，从而使用合适的配对方法。

GAP层可以与L2CAP建立联系，设置自定义的MTU值。

## 1. GAP角色
**GAP层定义了四种角色：**

- 外围设备（Peripheral）
- 中央设备（Central）
- 播报设备（Broadcaster）
- 观察设备（Observer）

外围设备可以发送广播数据和接收连接请求，进而建立连接，对应着链路层的广播状态。

中央设备扫描广播数据和发送连接请求，进而建立连接，对应着链路层的扫描状态或发起状态。

播报设备可以发送广播和接收扫描请求，通常不建立连接，对应着链路层的广播状态。

观察设备可以扫描广播和发起扫描请求，通常不建立连接，对应着链路层的扫描状态。

由于链路层支持同时拥有多个状态机，GAP层也支持一个设备同时具有多个GAP角色，比如在一个连接中充当中央设备，同时对外发出广播充当外围设备。

## 2.用户接口

GAP定义了几个与用户操作密切相关的参数：设备地址，设备名，PIN码和设备外观。

### 2.1 设备地址
设备地址在协议栈内部指BD_ADDR，在用户界面显示为“Bluetooth Device Address”。

设备地址为一个6字节的整形数组成，可以用冒号作为分隔符，比如00:0C:3E:3A:4B:69。

在用户界面，设备地址以自然顺序显示，而内部的BD_ADDR则以逆序保存，对于上述地址，BD_ADDR[0]等于0x69而不是0x00。

设备地址分为共有地址和随机地址，随机地址分为静态随机地址和私有地址，私有地址进一步分为可解析的私有地址和不可解析的私有地址。常用的地址为共有地址和可解析的私有地址两种类型。

关于设备地址类型分析，参见链路层的文章介绍。

### 2.2 设备名
设备名称仅起识别设备的作用，在用户界面显示为“Bluetooth Device Name”。

设备名最长可达248个字节，但是对端设备可能并不能显示这么长的名称。

设备名支持UTF-8编码，因此设备名可以使用中文。

### 2.3 PIN码
PIN码指两个设备配对时使用的passkey密码，在用户界面显示为“Bluetooth Passkey”。

PIN码为6位十进制整形数，因此它的有效范围为000000-999999（0x00000000 – 0x000F423F）。使用时必须显示全部6位数字，包括前导0。

### 2.4 设备外观
设备外观仅起到识别辅助别设备的作用，在用户界面显示为一个图标或一个字符串。

设备外观为一个2字节数，扫描设备可以通过设备外观值为设备分配一个合适的图标或描述。

## 3. 模式和规程
模式表示一种工作状态，规程是针对模式实现的一套操作方法。模式和规程成对出现，GAP规定了五种模式和规程，如下：

模式|规程
---|---
Broadcast Mode | Observer Procedure
Discovery Mode | Discovery Procedure
Connection Mode | Connection Procedure
Bonding Mode | Bonding Procedure
Periodic Advertising Mode | Periodic Advertising Procedure

### 3.1 播报模式和观察规程
播报模式并不等同于普通的广播状态。

在播报模式下，设备在广播事件中发送不可连接的广播数据，播报设备可以响应外部的扫描请求。

播报设备的广播数据格式与普通的广播数据相同，但是它不设置LE Limited Discoverable Mode和LE General Discoverable Mode这两个标志位，即播报设备是Non-discovery设备，这意味着中央设备扫描到该广播数据，应该选择忽略。

观察规程用于监听播报设备的广播数据和扫描响应数据，也可以监听普通广播设备。

### 3.2 可发现模式和发现规程
可发现模式分为三种，如下：

可发现模式 | 描述
---|---
Non-Discoverable mode | 不可发现模式，扫描设备应该选择忽略这类广播数据。
Limited Discoverable mode | 有限发现模式，广播数据仅工作有限时间内是可被发现的。
General Discoverable mod | 普通发现模式，没有额外限制。

有限发现模式将广播数据的LE Limited Discoverable Mode位置为1，普通发现模式将广播数据的LE General Discoverable Mode位置为1，如果这两个标志位均不设置，就是不可发现模式。

不可发现模式的广播数据与其他两种模式相同，所以其广播数据仍然能够被扫描设备正确读取，但由于没有设置相应的标志位，扫描设备在解析广播数据时应该尊重其不愿意被发现的意图，主动忽略该广播数据。

使用观察规程的观察设备，则不会忽略不可发现模式的广播数据。

有限发现模式通常用于用户指定的行为让设备临时进入可发现状态，可发现状态持续时间为T_GAP[lim_adv_timeout]。

普通发现模式是默认模式，它没有时间限制。

发现规程分为两种，如下：

发现规程 | 描述
---|---
Limited Discovery Procedure | 有限发现规程，仅能发现有限发现模式的广播数据。
General Discovery Procedure | 普通发现规程，没有额外限制。
有限发现规程，仅处理有限发现模式下的广播数据，包括设备地址和广播数据，忽略其他发现模式下的广播设备。

常规发现规程，能普通发现模式和有限发现模式下的广播数据。

此外还有一种发现规程，专用于发现设备名称，如下：

发现规程 | 描述
---|---
Name Discovery Procedure | 设备名发现规程，用于发现广播设备的设备名称。

设备名发现规程，可以发现普通发现模式和有限发现模式下的广播设备名称。

发现设备名称的步骤如下：

1. 建立连接
1. 读取GATT中的名字特征值

### 3.3 可连接模式和连接规程
可连接模式分三种，如下：

可连接模式 | 描述
---|---
Non-Connectable Mode | 不可连接模式，无法与其他设备建立连接。
Directed Connectable Mode | 定向可连接模式，可以与指定的中央设备建立连接。
Undirected Connectable Mode | 非定向可连接模式，可以与任何中央设备建立连接，这是默认的可连接模式。
而相关的连接规程则由四种，如下：

连接规程 | 描述
---|---
Auto Connection Establishment Procedure | 自动连接建立规程，利用中央设备的设备地址白名单，一旦地址匹配就自动建立连接。
General Connection Establishment Procedure | 普通连接建立规程，这是默认的连接规程，没有额外条件。
Selective Connection Establishment Procedur | 可选连接建立规程，利用中央设备的设备地址白名单，只有地址匹配的设备才能建立连接。
Direct Connection Establishment Procedure | 定向连接建立规程，与指定地址的外围设备建立连接。
此外，还有两个与连接相关的规程，如下：

规程 | 描述
---|---
Connection Parameter Update Procedure | 连接参数更新规程，更新连接参数信息。
Terminate Connection Procedure | 止连接规程，终止当前连接。
### 3.4 可绑定模式和绑定规程
可绑定模式分为：

可绑定模式 | 描述
---|---
Non-Bondable mode | 不可绑定模式，设备不支持配对操作，在配对请求命令中清除Bonding_Flags标志位。
Bondable mode | 可绑定模式，设备将设置认证请求命令中的Bonding_Flags标志位，并且保存绑定信息。

两个未绑定的设备，在访问需要绑定权限的数据时，执行绑定规程。

### 3.5 周期广播模式和周期广播规程
周期广播模式分为：

模式 | 描述
---|---
Periodic Advertising Synchronizability mode | 周期广播同步模式，发送周期广播事件的同步信息，适用于播报设备，
Periodic Advertising mode | 周期广播模式，发送周期广播数据，适用于播报设备

周期广播规程为：

规程 | 描述
---|---
Periodic Advertising Synchronization Establishment procedure | 周期广播同步建立规程，接收周期广播事件的同步信息并同步周期广播事件，适用于观察设备。
### 3.6 安全模式和认证规程
共有两种安全模式：

安全模式 | 描述
LE Security mode 1 | 安全模式1，使用认证信息保证安全。
LE Security mode 2 | 安全模式2，使用数字签名保证安全。
安全模式1下有四种安全级别：

- No security (No authentication and no encryption)
- Unauthenticated pairing with encryption
- Authenticated pairing with encryption
- Authenticated LE Secure Connections

四种安全级别围绕着认证和加密进行，安全级别依次增加，第1种安全级别没有认证和加密， 第2种安全基本提供未认证的加密，第3、4种安全级别能够提供认证和加密。

安全模式2下有两种安全级别：

- Unauthenticated pairing with data signing
- Authenticated pairing with data signing

假如设备同时要求加密和数字签名，将视认证需求选择合适的安全模式，比如需要认证则选择模式1.3，不需要认证则选择模式1.2，如果需要安全连接则选择模式1.4。

共有四种安全规程，如下：

规程 | 描述 | 适用安全模式
---|---|---
Authentication procedure | 认证规程，执行认证和加密操作。 | 安全模式1
Authorization procedure | 授权规程，用户行为确认是否为某个操作提供授权 | 安全模式1
Connection data signing procedure | 连接数据签名规程，在未加密的连接中传输认证的数据。 | 安全模式2
Authenticate signed data procedure | 认证已签名的数据规程，校验带有前面的数据是否有效。 | 安全模式2
Encryption procedure | 加密规程，对连接和数据进行加密。 | 安全模式1

### 3.7 隐私规程

隐私与私有地址有密切关系，跟私有地址相关的规程如下：

规程 | 描述
---|---
Non-resolvable private address generation procedure | 不可解析私有地址生成规程
Resolvable private address generation procedure | 可解析私有地址生成规程
Resolvable private address resolution procedure | 可解析私有地址解析规程
## 4. 广播包
广播包和扫描响应使用相同的数据格式，如下：

<center>![1-1](/images/blog/BLE/1-1.png)</center>

一个广播包由多个AD Structure组成，传统广播包的最大长度为31字节，扩展广播包的最大长度为255字节，未占用的数据则补零。

一个AD Structure中包含三个元素：长度、广播数据类型和广播数据。

其中长度指广播数据类型加上广播数据的总长度，广播数据类型决定了广播数据的属性，可以代表设备名、设备地址或服务的UUID。完整的广播数据类型可以在官方网站检索（链接）。

动态的广播数据适合放在广播包中发送，静态的广播数据适合放在扫描响应包中发送。

5. GAP特征项
每个BLE设备的GATT均包含必要的GAP服务项，GAP服务项包含以下特征项：

特征项 | UUID | 描述
---|---|---
Device Name	| 0x2A00 | 读取设备名称
Appearance | 0x2A01 | 读取设备外观
Peripheral Preferred Connection Parameters | 0x2A04 | 读取期望的连接参数
Central Address Resolution | 0x2AA6 | 中央设备支持解析地址，供外围设备读取以确定中央设备能否使用地址解析，仅在使能了隐私功能时使用，否则应删除
Resolvable Private Address Only | 0x2AC9 | 设备仅使用可解析的随机地址，供对端设备读取以确定该设备在绑定后是否仅使用可解析的随机地址，仅在使能了隐私功能时使用，否则应删除
