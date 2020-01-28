---
title : "认识BLE5协议栈[二]--安全管理层"
date : "2019-12-26T10:52:05+08:00"
tags : ["BLE5.0","协议栈","SM"]
series : ["BLE5协议栈"]
categories : ["无线BLE技术"]
draft : false
toc : true
---

安全管理（Security Manager）定义了设备间的配对过程。

配对过程包括了配对信息交换、生成密钥和交换密钥三个步骤。具有不同的输入输出能力的设备将采用不同的配对方式，两个设备完成配对将加密连接，产生LTK、IRK、CSRK等密钥，这些密钥将支持加密、隐私、签名等安全特性。
<!--more-->
安全管理协议定义了配对相关的数据结构。

安全管理数据都通过L2CAP的安全管理信道传输，安全管理协议通过GAP层暴露用户接口，由用户设置设备的输入输出能力和配对参数。

## 1. 配对概述
BLE 4.2协议新增了一种配对方法，称为“LE安全连接配对”，新的配对方法增加了安全性，为了与BLE4.1及以前的配对方法做区别，之前的配对方法统称为“传统配对”。

传统配对和LE安全连接配对过程基本一致，都分为三个步骤，仅第二步骤生成密钥上有所不同。

三个步骤为：

1. 交换配对信息，确定配对模式
1. 执行配对模式，生成密钥
1. 分发密钥，保存密钥

三个步骤具有明确的时序关系，并且前一个步骤将显著影响下一个步骤，如下所示：

<center>![2-1](/images/blog/BLE/2-1.png)</center>

两端设备先建立连接，然后才能进行配对操作。配对无需在连接后立即执行，可以在任何需要时候进行。

## 2. 配对信息
配对信息包括：
- 认证需求
- IO 能力
- OOB
- 密钥长度
- 是否绑定
### 2.1 安全特性
安全特性取决于设备的认证需求，可选的安全特性如下：

安全特性 | MITM保护 | 所属配对方法
---|---|---
LE Secure Connections pairing |	Yes | LE安全连接配对
Authenticated MITM protection | Yes | 传统配对
Unauthenticated no MITM protection | No | 传统配对
No security requirements | No | 传统配对

MITM（Man in the Middle）指中间人攻击，假如第三方设备攻破了BLE连接，A设备发送的消息被C设备接收，C设备再转发给B设备，A与B设备相互以为建立了连接，而实际上所有的数据通信都经过了C设备转发。

前两种安全特性可以实现MITM保护，后两种则无法防护MITM攻击。

四种安全等级从上至下安全性依次降低。

安全特性信息会持久保存在设备的安全数据库中。

### 2.2 IO能力
一个设备具有的输入输出能力分为以下几种情况：

输入能力 | 描述 | 输出能力 | 描述
---|---|---|---
No input | 无输入 | No ouput | 无输出
Yes/No | 仅能输入是或否 | Numeric output | 能显示数字
Keyboard | 输入数字以及确认 |  | 

不同的输入输出能力，将组合出不同的IO能力，如下：

No output |	Numeric| output
---|---|---
No input | NoInputNoOutput | DisplayOnly
Yes/No | NoInputNoOutput | DisplayYesNo
Keyboard | KeyboardOnly | KeyboardDisplay
### 2.3 OOB能力

OOB（Out of Band）指利用NFC或Wifi等非BLE通信方式传递密钥，它要求设备具有OOB接口能力。

传统配对方式中，要求两端设备都设定了OOB标志位，在LE安全连接配对方式中，只需一个设备设定了OOB标志位即可。

### 2.4 密钥长度
密钥长度决定了加密强度，越长的密钥其加密强度越高，但加解密所消耗资源和时间也越多。

密钥长度有效值为：7-16字节。

当两端设备的密钥长度值不同，取较小值为有效值。

## 3. 配对模式
可选的配对模式包括：

- Just Works
- Numeric Comparison
- Passkey Entry
- Out Of Band (OOB)

### 3.1 选择模式

如果两端设备均选择No-MITM protection安全特性，则使用Just Works配对模式；如果两端设备（对于LE安全连接配对，仅需要一个设备）均选择OOB，则使用OOB配对模式；否则根据两端设备的IO能力选择配对模式。

两端设备的IO能力中，如果有一端设备是NoInputNoOutput，则只能使用Just Works配对方式，其他的IO能力组合所对应的配对方式如下：

 NULL|DisplayOnly|DisplayYesNo|KeyboardOnly|KeyboardDisplay
 ---|---|---|---|---
DisplayOnly	|Just Works|Just Works|Passkey Entry|Passkey Entry
DisplayYesNo|Just Works|Just Works, Numeric Comparison|Passkey Entry|Passkey Entry, Numeric Comparison
KeyboardOnly|Passkey Entry|Passkey Entry|Passkey Entry|Passkey Entry
KeyboardDisplay|Passkey Entry|Just Works, Numeric Comparison	Passkey Entry|Passkey Entry, Numeric Comparison
如果采用了Just Works方式，一定是未认证的，Passkey Entry和Numeric Comparison方式则是认证的。

### 3.2 Just Works
Just Works配对模式不能够防护窃听和MITM威胁。如果设备的配对过程不被窃听，配对结束后连接被加密并且保证安全。

所以Just Works虽然不如其他配对方式安全，仍然比不配对的连接要安全。

Just Works通常用于一端设备完全没有输入输出能力的场景，所以它将临时密钥TK设置为固定值0，这样两端设备可以根据这个TK值进行后续的加密操作。

### 3.3 Passkey Entry
Passkey Entry配对模式可以防护MITM威胁，有限的防护窃听。如果设备的配对过程不被窃听，配对结束后连接被加密并且保证安全。

Passkey Entry方法在一端设备上显示一个随机的6位十进制数作为密码，另一端设备输入该密码。passkey作为TK的初值进行后续加密运算，进而可以通过比较加密运算的中间值来判断输入的密码与显示的密码是否一致。比如passkey=001024(400h)，则TK=0x00000000000000000000000000000400。

### 3.4 OOB
OOB（Out of Band）配对模式使用非BLE协议传输TK。用户通过键盘输入Passkey Entry，也属于一种OOB传输TK。

OOB传输通道的安全性决定了配对过程的安全性。

### 3.5 Numeric Comparison
数值比较仅能用于LE安全连接配对方法，它能够防护MITM和窃听威胁。

数值比较配对模式在两端设备分别显示一串数字，用户比较数字是否相等并通过输入Yes/No来确认，从而实现认证。

这种模式仅需要两个按键即可完成输入，适合用在小型设备上。

### 3.6 安全性
BLE通信面临的外部威胁有两类：被动威胁和主动威胁。

被动威胁指第三方设备监听配对过程中的密钥数据，有了密钥即可解密后续的连接数据。

主动威胁指MITM，通过伪造身份参与到通信连接中。

一旦BLE的连接经过了认证，即可抵挡MITM威胁，经过认证的设备，就意味着对端设备不是一个伪造偷听设备。

传统的配对方法使用临时密钥TK作为加密运算的初值，而交换TK时可以被窃听设备获取，从而破解后续的加密措施。

不同的配对方法，其安全防护能力如下：

安全性 | 传统配对 | LE安全连接配对
---|---|---
MITM防护 | Passkey Entry， OOB | Numeric Comparison， Passkey Entry， OOB
窃听防护 | OOB | 全部

## 4. 生成密钥
### 4.1 传统配对
传统配对可以使用Just Works，Passkey Entry和OOB三种算法，配对成功后生成短期密钥STK（Short Term Key）。

生成STK的算法如下：

STK = s1(TK, Srand, Mrand)

其中s1算法是专门用于生成STK的加密算法，TK值可以通过不同的配对模式获得（Just Works模式下TK=0，Passkey Entry模式下TK=passkey，OOB模式下TK=oob input），Mrand和Srand分别为主机和从机生成的128位的随机数。

在两端设备生成STK之前，还需要确认对端设备的安全性，称为“认证”。

配对发起端设备生成一个Mrand随机数，然后根据TK值以及设备配对信息和地址信息等，生成一个确认值Mconfirm。配对响应端设备也按照相同的步骤生成一个Srand和Sconfirm。

然后两端设备交换各自的随机数和确认值。

响应端根据发起端的随机数Mrand按照相同的算法计算出一个新的Mconfirm_new，比较Mconfirm和Mconfirm_new，如果二者匹配，说明发起端设备是安全的。

发起端根据发起端的随机数Srand计算出新的Sconfirm_new，比较Sconfirm和Sconfirm_new，如果二者匹配，说明响应端设备是安全的。

然后两端设备命令控制器加密连接，并根据Srand和Mrand生成STK值。

一旦产生了STK，即说明两端设备之间的连接已经加密和认证。

### 4.2 LE安全连接配对
安全连接配对可以使用全部四种配对模式，配对成功后生成长期密钥LTK（Long Term Key）。

安全连接配对模式使用椭圆曲线加密算法ECDH（ Elliptic Curve Diffie-Hellman）来解决TK被窃听的威胁。

ECDH算法具有数学不可逆的特点。对于一个ECDH运算Q=Pk，如果P和k是已知，计算出Q很容易，但是反过来如果P和Q是已知，计算出k则很难。在密钥交换时，设置一对“公钥-私钥”对，私钥是上式子中的k，公钥是Q，加密算法为P，交换密钥时仅交换公钥Q，私钥永远不对外暴露，攻击者即使获得了公钥也无法反推出私钥。（参考）

引入ECDH算法是安全连接配对与传统配对最大的区别。

生成LTK需要三个步骤：

1. 交换公钥，生成ECDH密钥
1. 认证设备
1. 利用私钥生成LTK
在配对开始前，每个设备先准备自己的公钥和私钥对，公钥可以对外传输，私钥不对外传输。

第一步交换攻击，并利用公钥和私钥生成ECDH密钥，这个ECDH密钥将用来生成LTK，由于私钥和ECDH密钥不对外传输，因此可以保证不被窃听。

第二步认证对端设备，具体过程与传统配对基本一致，即在设备内根据随机数生成一个确认值，再将随机数和确认值穿给对端设备，对端设备根据随机数重新生成一个确认值，并与收到的确认值作比较，符合则认证通过，不符合则认证失败。

如果使用Just Works配对模式，无需用户输入，两端设备自动完成随机数和确认值的交换和校验过程，完成认证。

如果使用Numeric Comparison配对模式，无需用户输入，两端设备自动完成随机数和确认值的交换过程，并将最终确认值显示出来等待用户确认，如果确认通过则完成认证。

如果使用Passkey Entry配对模式，需要用户输入passkey作为初值，然后两端设备自动完成随机数和确认值的交换和校验过程，完成认证。

如果使用OOB配对模式，不强制要求双向OOB通信，仅单向OOB通信即可完成认证。利用OOB通道传输随机数，保证随机数不被窃听。

第三步生成LTK，仍然要执行一次确认值校验操作。

## 5. 分发密钥
BLE工作时可能用到以下几种密钥：

密钥 | 描述 | 适用范围
---|---|---
IRK (Identity Resolving Key) | 身份识别密钥，用于解析私有地址	| 传统配对、LE安全连接配对  
CSRK (Connection Signature Resolving Key) | 连接签名解析密钥，用于解析签名数据 | 传统配对、LE安全连接配对
LTK (Long Term Key) | 长期密钥，用于解析加密连接。传统配对中，LTK是由STK进一步生成；LE安全连接配对中，LTK是配对结束后生成 | 传统配对
EDIV (Encrypted Diversifier) | 在传统配对中，用于生成LTK。它是绑定信息的一部分。 | 传统配对
Rand (Random Number) | 在传统配对中，用于生成LTK。它是绑定信息的一部分。 | 传统配对

其中Rand用以生成EDIV，EDIV用以生成LTK。传统配对的第二步骤结束时，连接被STK加密以传输各种密钥，此时将STK临时当做LTK使用，配对成功以后，需要使用Rand、EDIV和LTK作为密钥来加密和解析连接。

加密过程在链路层进行，通过执行链路层的加密规程进行加密。查看LL_ENC_REQ命令，其输入参数包括Rand和EDIV。

密钥生成以后，两端设备将交换各自的密钥信息。传统配对设备可以交换以上全部五种密钥，LE安全连接配对仅交换IRK和CSRK。交换密钥时会保存对端设的设备地址，使设备地址与LTK关联在一起。

至此，配对过程全部结束。

## 6. 安全管理协议
安全管理协议定义配对过程中用到的各种数据格式和协议接口。

安全管理操作数据使用L2CAP的Security Manager信道，传统的配对方法使用默认的L2CAP MTU值23，LE安全连接配对方法将L2CAP MTU值扩大到65。

安全管理协议的操作以命令的形式进行，命令的格式如下：

字段 | Code | Data
---|---|---
长度 | 1 octets | 0 – 22 or 64 octets
命令码Code代表了不同的命令类型，全部命令码如下：

Code |	Command
  ---|---
0x01 |	Pairing Request
0x02 |	Pairing Response
0x03 |	Pairing Confirm
0x04 |	Pairing Random
0x05 |	Pairing Failed
0x06 |	Encryption Information
0x07 |	Master Identification
0x08 |	Identity Information
0x09 |	Identity Address Information
0x0A |	Signing Information
0x0B |	Security Request
0x0C |	Pairing Public Key
0x0D |	Pairing DHKey Check
0x0E |	Pairing Keypress Notification
下面介绍配对请求的命令格式，配对响应与之完全相同。

发起端设备发起配对请求，执行配对信息交换（Pairing Feature Exchange）。

配对请求命令的结构如下：

<center>![2-2：SM_Pair_Request](/images/blog/BLE/2-2.png)</center>

- IO Capability表示不同的IO能力
- OOB data flag表示是否具有OOB能力

AuthReq表示认证请求，它的结构如下：
字段 | Bonding_Flags | MITM | SC | Keypress | CT2 | RFU
---|---|---|---|---|---|---
长度 | 2 bits | 1 bit | 1 bit | 1 bit | 1 bit | 2 bits

其中Bonding_Flags表示是否保存绑定信息，MITM是否要求MITM防护能力，即是否需要认证，SC（Secure Connection）表示是否使用LE安全连接配对，Keypress为KeyboardOnly设备提供一些必要的输入状态信息，CT2与经典蓝牙有关。

- Max Encryption Key Size表示最大密钥长度，有效范围为7-16字节。
- Initiator Key Distribution表示发起者需要分发的密钥。
- Responder Key Distribution表示响应者需要分发的密钥。
