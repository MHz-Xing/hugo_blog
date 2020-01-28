---
title : "认识BLE5协议栈[四]--属性协议层"
date : "2019-12-26T11:05:41+08:00"
tags : ["BLE5.0","协议栈","ATT"]
series : ["BLE5协议栈"]
categories : ["无线BLE技术"]
draft : false
toc : true
---

属性协议（Attribute Protocol）简称ATT。

ATT层定义了属性实体的概念，包括UUID、句柄和属性值等，也规定了属性的读、写、通知等操作方法和细节，这些与属性操作相关的内容称为属性协议。ATT层规定了ATT_MTU值，如果属性值很长，超过了ATT_MTU限制，将使用特殊的读写方法进行操作。

基于ATT层，可以构建出通用属性操作规范。

<!--more-->

## 1. 属性
在蓝牙协议中， 属性是指一个数据实体，它包含标识符，句柄，数据内容，访问权限，安全问题等。

属性协议规定了属性的发现和读写访问的方法。

### 1.1 类型
属性类型由一个UUID（Universally Unique Identifier）表示。UUID是指从时间尺度和空间尺度都具有唯一性的一串128-bit的数字，该数字串在全球范围内不会重复，并且在未来也不会出现重复。

一个典型的16字节UUID格式为XXXX-XX-XX-XX-XXXXXX。

蓝牙协议设定了一个蓝牙基础UUID： 00000000 – 0000 – 1000 – 8000 – 00805F9B34FB。

利用该基础UUID，可以使用16-bit或32-bit的UUID来代替128-bit的UUID，当传递到对端设备，再还原成128-bit的UUID。

假如16-bit的UUID为YYYY，则还原后的128-bit的UUID为：0000YYYY – 0000 – 1000 – 8000 – 00805F9B34FB。

假如32-bit的UUID为YYYYYYYY，则还原后的128-bit的UUID为：YYYYYYYY – 0000 – 1000 – 8000 – 00805F9B34FB。

ATT层支持使用16-bit和128-bit两种UUID，32-bit的UUID在使用前必须转换成128-bit。

### 1.2 句柄
属性句柄犹如指向属性实体的指针，对端设备通过句柄来访问该属性。

属性句柄是一个2字节数，有效范围为0x0001-0xFFFF。

属性句柄为有序排列，后面的句柄值会大于前面的句柄，通常下一个属性的句柄值是上一个属性的句柄加1。

### 1.3 分组
多个属性可以合成一组，一组属性包含的参数为：开始句柄、结束句柄。

### 1.4 值
属性值可以是一个数字或一个字符串。属性值的长度信息不包含在PDU中，所以需要从PDU的长度间接推算出属性值的长度信息。

属性值通常在一个PDU中发送，如果属性值太长，也可以分成多个PDU进行发送。

### 1.5 权限
属性的读写权限由ATT层之上的协议层规定，有效的读写权限包括：可读、可写、读写。

在读写属性之前，客户端设备还需要具有足够的安全权限，包括：

- 加密权限：需要加密、无需加密
- 认证权限：需要认证、无需认证
- 授权权限：需要授权、无需授权

如果权限不足，将触发错误处理机制。

### 1.6 控制点属性
控制点属性是一类特殊属性，它不可读，只能写。

### 1.7 协议方法
对属性的操作称为协议方法，包括：命令（Command），请求（Request），响应（Response），通知（Notification），指示（Indication）和确认（Confirmation），某些属性PDU还涉及授权签名方法。

### 1.8 交换MTU Size
ATT_MTU表示ATT层间传输的数据包的最大长度。两端设备可以通过Exchange MTU Request/Response进行交换MTU。

### 1.9 长包属性
对于读属性的数据包，最大长度为（ATT_MTU-1）个字节。其中减去的1表示1字节的操作码。

对于写属性的数据包，最大长度为（ATT_MTU-3）个字节。其中减去的3表示1字节的操作码和2字节的属性句柄。

如果数据包超过这个长度，则称为长包属性。

读长包属性，需要使用Read Blob Request，写长包属性，需要使用Prepare Write Request和Execute Write Request。

如果使用普通Read操作读长包属性，仅能读取前（ATT_MTU – 1）个字节，使用普通Write操作写长包属性，仅能写前（ATT_MTU-3）个字节。

无论普通属性还是长包属性，属性值的最大长度均为512字节。

### 1.10 原子操作
一个请求或一个命令，称为一个原子操作。一个原子操作结束后，才能进行新的原子操作。

读写长包属性无法在一个原子操作内完成。

## 2. 属性PDU
### 2.1 属性角色
在ATT层协议框架内，拥有一组属性的设备称为服务端（Server），读写该属性值的设备称为客户端（Client）。

### 2.2 分类
属性PDU有六类：

属性PDU | 方向 | 触发响应
---|---|---
Command | Client -> Server | –
Request | Client -> Server | Response
Response | Server -> Client | –
Notification | Server -> Client | –
Indication | Server -> Client | Confirmation
Confirmation | Client -> Server | –

属性PDU格式如下：

字段 | Opcode | Parameter | Authentication Signature
---|---|---|---
长度 | 1 octet | 0 – (ATT_MTU-X) octets | 0 or 12 octets

其中Opcode的第0-5位表示该属性的具体类型，第6位表示命令标志位，如果该位为1，表示该操作码对应一个命令，最后1位表示认证签名（Authentication Signature）标志位，如果该位为1，表示该PDU的最后一个字段中包含12字节的认证签名。

Parameter字段中包含了参数，其长度为0支ATT_MTU-x，如果认证签名位为1，则此处x等于13，否则等于1。

只有写命令才需要认证签名，其他命令不需要。此外，如果链路已经进行加密，则属性PDU中也无需额外添加认证签名。

### 2.3 操作顺序
对于Request和Indication属性，需要接收端返回响应。在发出Request和Indication后，收到响应之前，不能发出新的Request和Indication。

对于其他无需响应的属性，则可以在自由发送，但是不保证接收端一定能够收到和执行。

可以在Request和Response之间，或Indication和Confirmation之间发送其他无需响应的属性。

### 2.4 事务
一个Request-Response对，或Indication-Confirmation对，称为一个事务。

对于客户端设备而言，发出Request或收到Indication表示事务的开始，收到Response或返回Confirmation表示事务的结束。

对于服务端设备而言，发出Indication或收到Confirmation表示事务的开始，收到Confirmation或返回Response表示事务的结束。

## 3. 属性协议PDU
属性协议规定了多种Request-Response对，请求属性由客户端设备发出，响应属性由服务端设备发出。

### 3.1 错误处理
Opcode | PDU
---|---
0x01 | Error Response

如果属性PDU的操作码无效，或属性句柄无效，将返回错误响应PDU。在PDU的Parameter字段中，包含了错误编码。

### 3.2 交换MTU
Opcode | PDU
---|---
0x02 | Exchange MTU Request
0x03 | Exchange MTU Response

客户端设备向服务端设备发送交换MTU请求，提供客户端设备的MTU值。服务端设备获知客户端的MTU值，并返回自己的MTU值。两端设备都将设置较小的MTU值作为新的MTU值。

如果两端设备没有交换MTU，则使用默认的MTU值（BLE下为23）处理属性事务。

### 3.3 查找信息
PDU | Opcode
---|---
0x04 | Find Information Request
0x05 | Find Information Response
0x06 | Find By Type Value Request
0x07 | Find By Type Value Response

查找信息请求，包含两个参数：起始属性句柄和结束属性句柄，用于获取服务端设备属性句柄处于该参数区间内的属性。

查找信息响应，包含指定句柄区间内的属性UUID。如果区间内有多个属性，则返回多个响应。

按类型值查找请求，是在查找信息请求的基础上，加上了属性类型和属性值两个参数，这样能够更加精确的找到目标属性。

按类型值查找响应，包含了满足条件的属性句柄列表。

### 3.4 读属性
Opcode | PDU
---|---
0x08 | Read By Type Request
0x09 | Read By Type Response
0x0A | Read Request
0x0B | Read Response
0x0C | Read Blob Request
0x0D | Read Blob Response
0x0E | Read Multiple Request
0x0F | Read Multiple Response
0x10 | Read by Group Type Request
0x11 | Read by Group Type Response

按类型读请求，包含三个参数：起始属性句柄、结束属性句柄和属性类型。

按类型读响应，包含了满足条件的属性的“句柄-值”对的列表。

读请求，包含一个参数：属性句柄。

读响应，返回满足条件的属性值。

读片段（blob）请求，用于读取一个长包属性的值，它包含两个参数：属性句柄和偏移量。以不同的偏移量作为参数，多次执行该请求可以读取长包属性的完整值。

读片段响应，包含了长包属性值的指定偏移量片段。

读多次请求，用于读取多个给定句柄的属性值，它包含一个参数：句柄列表。

读多次响应，包含了多个指定句柄的属性值。

按组类型读请求，用于读取指定组类型的属性值，组类型是由ATT层之上的协议层设定的。它包含三个参数：起始属性句柄、结束属性句柄和属性组类型。

按组类型读响应，包含了满足条件的属性值列表。

### 3.5 写属性
Opcode | PDU
---|---
0x12 | Write Request
0x13 | Write Response
0x14 | Write Command
0x15 | Signed Write Command
写请求，将待写数值写入指定的属性值，包含两个参数：属性句柄和数值。

写响应，表示写请求执行成功，不含任何参数。

写命令，将待写数值写入指定的属性值，包含两个参数：属性句柄和数值。它不会触发一个写响应。

签名的写命令，与上面的写命令类似，指示包含了额外的参数：认证签名。典型应用是写控制点属性。

### 3.6 队列写属性
队列写是指利用一个先进先出的队列，缓存多个属性值的写操作，然后在一个原子操作中完成所有的值写入操作。

队列写专门用于长包属性的写操作，现将一个长数据分成多个部分并记录偏移量，然后通过队列缓存，等数据发送完毕，再按照收到的顺序，一次性将整个长数据写入属性值。

Opcode | PDU
---|---
0x16 | Prepare Write Request
0x17 | Prepare Write Response
0x18 | Execute Write Request
0x19 | Execute Write Response
准备写请求，用于发送一个长数据片段，它包含三个参数：属性句柄、偏移量和待写入数据。

准备写响应，收到准备写请求以后，缓存收到的数据。

执行写请求，对前面缓存的数据执行写操作，它包含一个参数：标志位。如果标志位为1，则执行写操作，如果为0，则取消前面的缓存数据。

执行写响应，根据执行写请求的标志位，执行或取消写操作。

### 3.7 通知属性
Opcode | PDU
---|---
0x1B | Handle Value Notification
0x1D | Handle Value Indication
0x1E | Handle Value Confirmation

发送数值通知，它包含两个参数：属性句柄和属性值。它不需要客户端收到后返回响应。

发送数值指示，它包含两个参数：属性句柄和属性值。它需要客户端收到后返回确认。

发送数值确认，它不包含参数，客户端发出该确认消息表示收到了数值指示。

## 4. 权限
通知和指示与读写操作类似，也可以设置安全权限。

每个属性可以设置单独的权限。

权限不足将阻止操作，并触发错误响应。

权限问题与ATT之上的协议层有较大联系。
