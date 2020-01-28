---
title : "认识BLE5协议栈[六]--主机控制器接口"
date : "2019-12-26T11:24:30+08:00"
tags : ["BLE5.0","协议栈","HCI"]
series : ["BLE5协议栈"]
categories : ["无线BLE技术"]
draft : false
toc : true
---

BLE协议栈规定物理层、链路层和DTM层属于控制器，其他协议层属于主机，主机与控制器之间的通信是通过主机控制器接口传输层完成的。

主机控制器接口常简称为HCI（Host Controller Interface）。

HCI定义了一套“命令-事件”机制，主机向控制器发送HCI命令，控制器向主机返回命令执行结果。应用层的所有操作都会转换成HCI命令传给控制器。

<!--more-->

## 1. HCI通信
HCI接口物理形式可以是串口、SPI、USB和三线串口。

对于串口HCI，其通信模型如下：

<center>![6-1：HCI_UART_Interface](/images/blog/BLE/6-1.png)</center>

左侧蓝牙主机向右侧蓝牙控制器发送命令，控制器返回命令执行状态。当收到对端设备发送的消息，控制器会以事件形式发送给主机。

通过HCI的数据包括：HCI命令、HCI事件和连接数据。HCI层本身不能区分这三种类型，因此在发送HCI数据包前需要先发送该数据包的类型指示信息。串口HCI的数据包类型指示信息如下：

HCI包类型 | 指示信息
---|---
HCI命令 | 0x01
连接数据 |0x02
HCI事件 |0x04

指示信息中缺少0x03，该信息用于经典蓝牙概念。

包类型指示位在HCI包发送前发给给主机或控制器。

## 2. 连接数据

两个设备建立连接后相互收发数据，从主机将数据发送给控制器，再通过无线发送到对端设备，或控制器接收到对端设备数据后通过HCI发送给主机。连接数据的结构如下所示：

<center>![6-2：HCI_Data_Packet_Format](/images/blog/BLE/6-2.png)</center>

- Handle：连接句柄。
- FB Flag：数据边界标志（Packet Boundary Flag），表示当前数据包是一个完整数据包的开头片段或中间片段。
- BC Flag：播报标志（Broadcast Flag），不用于BLE。
- Data Total Length：数据总长度。
- Data：有效数据。
- 
## 3. HCI命令
HCI命令包包括：操作码OpCode、参数总长度和参数个数，如下所示：

<center>![6-3：HCI_Command_Packet_Format](/images/blog/BLE/6-3.png)</center>

为了避免控制器的缓冲区溢出，发送命令包时需要应用流程控制。主机向控制器发送一个命令，控制器返回命令执行状态事件，事件中包含参数Num HCI Command Packets，该参数指主机可以发送的最大命令包的数量。

控制器按接收顺序执行主机命令，但后面的命令可能提前执行完毕。

如果命令执行出错，将在控制器的状态事件中包含错误码。

HCI命令非常多，将近300个，BLE仅支持部分命令，所有BLE专属的命令OGF字段都等于0x08。

下面列出BLE支持的最基础的一部分HCI命令：

命令 |描述
---|---
LE Add Device To White List Command | 添加白名单
LE Clear White List Command	| 清空白名单
LE Read Buffer Size Command	| 读控制器缓存
LE Read Local Supported Features Command | 读本设备支持的功能
LE Read Supported States Command | 读本设备支持的状态
LE Read White List Size Command	| 读白名单空间
LE Remove Device From White List Command | 从白名单移除设备
LE Set Event Mask Command | 设置事件掩码
LE Test End Command | 结束测试
Read BD_ADDR Command | 读取设备地址
Reset Command | 重启
LE Read Advertising Channel TX Power Command | 读取广播发射功率
LE Transmitter Test Command	| 发送数据测试
LE Set Advertising Data Command	| 设置广播数据
LE Set Advertising Enable Command | 开启广播
LE Set Advertising Parameters Command | 设置广播参数
LE Set Random Address Command | 设置随机地址
LE Receiver Test Command | 接收数据测试
LE Set Scan Enable Command | 开启扫描
LE Set Scan Parameters Command | 设置扫描参数
Disconnect Command | 断开连接

## 4. HCI事件
HCI事件包包括：时间代码， 参数总长度和具体参数，如下所示：

<center>![6-4：HCI_Event_Packet_Format](/images/blog/BLE/6-4.png)</center>

HCI事件包不强制要求流程控制，因为通常主机总是具有充足资源来处理控制器返回的事件。

当连接断开时，主机默认所有命令都已经执行完毕，将不再接收任何事件。

控制器收到不同的主机命令，可能返回以下类型事件：

- 执行完毕事件
- 状态信息事件

对于不涉及连接的命令，可以立即得到执行结果，执行完毕事件报告该命令执行成功或失败。

对于涉及连接的命令，无法立即得到执行结果，命令执行完毕后，先返回执行完毕事件，等命令最终结果产生，再返回新的执行完毕事件。比如LE Create Connection Command命令，执行命令时先返回执行完毕，表面链路层开始执行或加入执行队列，待两端设备建立连接，将返回连接完成事件。

部分读命令，比如LE Read Advertising Channel Tx Power Command，执行完毕后将读取结果存放在状态信息事件中返回。

HCI事件包括BLE专有事件和通用事件，通用事件适用于经典蓝牙和BLE。BLE专有事件称为“元事件（LE Meta Event）”，共有20个，它们的事件代码均为0x3E，事件参数的第一个字节为Subevent_code，用以区分不同的元事件。如下：

事件 | Subevent_Code | 描述
---|---|---
LE Connection Complete Event | 0x01 | 建立连接完毕
LE Advertising Report Event | 	0x02 | 检测到广播数据或收到扫描响应数据
LE Connection Update Complete Event | 0x03 | 连接参数更新完毕
LE Read Remote Features Complete Event | 0x04 | 读取对端设备功能完毕
LE Long Term Key Request Event | 0x05 | 控制器向主机发送LTK以加密链接
LE Remote Connection Parameter Request Event | 0x06 | 对端设备发起更新连接参数请求
LE Data Length Change Event | 0x07 | 控制器通知主机链路层数据长度发生了更新
LE Read Local P-256 Public Key Complete Event | 0x08 | 控制器通知主机P-256密钥生成完毕
LE Generate DHKey Complete Event | 0x09 | 控制器通知主机椭圆加密算法密钥生成完毕
LE Enhanced Connection Complete Event | 0x0A | 建立连接完毕（还支持扩展连接）
LE Directed Advertising Report Event | 0x0B | 检测到定向广播数据或扫描响应数据
LE PHY Update Complete Event | 0x0C | 物理层更新完毕
LE Extended Advertising Report Event |0x0D |检测到扩展广播数据或扫描响应数据
LE Periodic Advertising Sync Established Event | 0x0E | 建立周期广播同步完毕
LE Periodic Advertising Report Event | 0x0F | 检测到周期广播数据或扫描响应数据
LE Periodic Advertising Sync Lost Event | 0x10 | 周期广播数据无法同步
LE Scan Timeout Event |0x11 | 扫描超时
LE Advertising Set Terminated Event | 0x12 | 终止广播数据集事件
LE Scan Request Received Event | 0x13 | 收到扫描请求
LE Channel Selection Algorithm Event | 0x14 | 使用了信道选择算法
