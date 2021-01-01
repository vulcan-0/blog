---
title: Wireshark系列（1） —— 数据包分析基础
date: 2020-12-29 20:06:08
tags:
    - 网络
    - Wireshark
categories: 网络
---

本文以我们相对比较熟悉的HTTP协议为例子，对Wireshark抓取到的数据包进行分析。在开始本文之前，假定读者已经掌握了如何通过Wireshark进行抓包。

# 总览

![](/images/wireshark/Wireshark.png)

上图展示的是一个通过Wireshark抓取到的数据包，下面针对此包，结合网络各层协议进行详细分析。

# Frame

![](/images/wireshark/Wireshark-Frame.png)

```javascript
Frame 4465: 171 bytes on wire (1368 bits), 171 bytes captured (1368 bits) on interface en5, id 0
    Interface id: 0 (en5)
        Interface name: en5
        Interface description: AX88x72A
```

其中，帧序号为`4465`，一共捕获了`171字节`（`1368位`）数据，网络接口ID为`0`，名称为`en5`，网卡名称为`AX88x72A`。各部分的详细含义如下：

```javascript
Encapsulation type: Ethernet (1) // 封装类型为：以太网
Arrival Time: Dec 29, 2020 20:05:42.808789000 CST // 捕获时间
[Time shift for this packet: 0.000000000 seconds] // 时间偏移量，固定为：0.000000000
Epoch Time: 1609243542.808789000 seconds // 新纪元时间，从标准世界时1970年1月1日0时0分0秒起到现在的总秒数
[Time delta from previous captured frame: 0.000258000 seconds] // 此包与前一个被捕获的包的时间间隔
[Time delta from previous displayed frame: 0.000000000 seconds] // 此包与前一个显示的包的时间间隔
[Time since reference or first frame: 103.353277000 seconds] // 此包与第一帧的时间间隔
Frame Number: 4465 // 帧序号，捕获到的第一帧的序号为1
Frame Length: 171 bytes (1368 bits) // 帧长度
Capture Length: 171 bytes (1368 bits) // 捕获字节数
[Frame is marked: False] // 是否做了标记（右键->Mark/Unmark Package）
[Frame is ignored: False] // 是否忽略了此包（右键->Ignore/Unignore Package）
[Protocols in frame: eth:ethertype:ip:tcp:http] // 帧内封装的协议
[Coloring Rule Name: HTTP] // 着色标记的协议名称
[Coloring Rule String: http || tcp.port == 80 || http2] // 着色标记的字符串
```

# Ethernet

![](/images/wireshark/Wireshark-Ethernet.png)

![](/images/wireshark/Wireshark-Ethernet-2.png)

```javascript
Ethernet II, Src: 00:00:00_00:21:a7 (00:00:00:00:21:a7), Dst: HuaweiTe_9e:52:23 (44:a1:91:9e:52:23)
```

`Ethernet II`是数据链路层的以太网协议。目的物理地址：`44:a1:91:9e:52:23`，源物理地址：`00:00:00:00:21:a7`，对应到上图展示的传输字节码中的前面0-11个字节。第12、13个字节表示以太类型，`0x0800`表示封装的上层协议为`IPv4`。以太网帧的协议格式如下：

![](/images/wireshark/Wireshark-Ethernet-3.png)

# IPv4

![](/images/wireshark/Wireshark-IPv4.png)

![](/images/wireshark/Wireshark-IPv4-2.png)

![](/images/wireshark/Wireshark-IPv4-3.png)

该层为网络层，对照着IPv4的协议格式，IP协议头中的第0个字节（`0x45`）由两部分组成，前面4位表示版本为`4`；后面4位表示首部长度为`5`，说明首部有多少个32位字（4字节），相当于5*4=20字节。

第1个字节的前6位表示区分服务类型，一般不使用，二进制值一般为`000000`。后面2位表示显式拥塞通告（ENC），在[RFC 3168](https://datatracker.ietf.org/doc/rfc3168/)中定义，允许在不丢弃报文的同时通知对方网络拥塞的发生。ECN是一种可选的功能，仅当两端都支持并希望使用，且底层网络支持时才被使用。

第2、3个字节表示报文总长度为`157`（16进制为`0x009d`），加上数据链路层的`14`字节，长度刚好等于`171`字节。

第4、5个字节用来唯一地标识一个报文中的所有分片，没有分片时，值为：`0x0000`

第6、7个字节的前3位为标识位，其中第0位为保留位，固定为0；第1位，当值为0时表示允许分片，为1时表示禁止分片；第2位为0表示已经是最后一个分片，为1表示后面还有分片。后面13位指明了每个分片相对于原始报文开头的偏移量，以8字节作单位。此处的值为`0x4000`，即二进制的`0100 0000 0000 0000`，即禁止分片，分片偏移量为0。

第8个字节表示存活时间，`0x40`即`64`。存活时间以秒为单位，但小于1秒的时间均向上取整到1秒。在现实中，这实际上成了一个跳数计数器：报文经过的每个路由器都将此字段减1，当此字段等于0时，报文不再向下一跳传送并被丢弃，最大值是255。第9个字节表示传输层协议，`0x06`表示协议为`TCP`。

第10、11个字节表示首部校验和，此处值为`0xb8b3`。

第12-15个字节表示的是源IP地址`0xac144d47`（即`172.20.77.71`），第16-19个字节表示的是目的IP地址`0xa83ee00d`（即`168.62.224.13`）。

# TCP

![](/images/wireshark/Wireshark-TCP.png)

![](/images/wireshark/Wireshark-TCP-2.png)

![](/images/wireshark/Wireshark-TCP-3.png)

TCP为网络层协议，对照着协议格式看，可以知道第0、1个字节表示源端口，`0xc4ea`即`50410`，第2、3个字节表示目的端口，`0x0050`即`80`。

Stream index，是Wireshark定义的一个字段，用来标识一个TCP会话连接，我们选择右键->Follow->TCP Stream时，就是通过这个index将整个会话的TCP报文串联在一起的。

TCP Segment Len，表示TCP报文的长度，指的是除去TCP首部之外的报文长度，这里为`105`，这个数字是怎么来的？下面会进行说明。

第4-7个字节表示序列号，`0x82f01bff`即`2196773887`，第8-11个字节表示确认号，`0x4374d387`即`1131729799`。

```javascript
Sequence number: 1    (relative sequence number)
[Next sequence number: 106    (relative sequence number)]
Acknowledgment number: 1    (relative ack number)
```

这几个`relative number`都是在Wireshark中定义的相对序号，它们是怎么来的以及有什么作用？后面在讲述TCP握手与挥手的文章中会详细展开。

第12、13个字节中，前4位表示数据偏移，即从哪个字节开始是数据，因此也可以将其理解为TCP首部的长度，以4字节为单位，`0x8`，即首部长度为：8*4=32字节。我们前面已经知道IP报文总长度为157，IP首部长度为20，因此TCP报文总长度为137，其中TCP首部长度为32，因此TCP数据长度为105，这就解析了前面的`TCP Segment Len`的来源了。接着后面3位是保留为，值均为0。最后面9位从前往后分别表示：

- NS — ECN-nonce。ECN显式拥塞通知（Explicit Congestion Notification）是对TCP的扩展，定义于[RFC 3540（2003）](https://datatracker.ietf.org/doc/rfc3540/)。ECN允许拥塞控制的端对端通知而避免丢包。ECN为一项可选功能，如果底层网络设施支持，则可能被启用ECN的两个端点使用。在ECN成功协商的情况下，ECN感知路由器可以在IP头中设置一个标记来代替丢弃数据包，以标明阻塞即将发生。数据包的接收端回应发送端的表示，降低其传输速率，就如同在往常中检测到包丢失那样。
- CWR — Congestion Window Reduced，定义于[RFC 3168（2001）](https://datatracker.ietf.org/doc/rfc3168/)。
- ECE — ECN-Echo有两种意思，取决于SYN标志的值，定义于[RFC 3168（2001）](https://datatracker.ietf.org/doc/rfc3168/)。
- URG — 为1表示高优先级数据包，紧急指针字段有效。
- ACK — 为1表示确认号字段有效。
- PSH — 为1表示是带有PUSH标志的数据，指示接收方应该尽快将这个报文段交给应用层而不用等待缓冲区装满。
- RST — 为1表示出现严重差错。可能需要重新创建TCP连接。还可以用于拒绝非法的报文段和拒绝连接请求。
- SYN — 为1表示这是连接请求或是连接接受请求，用于创建连接和使顺序号同步。
- FIN — 为1表示发送方没有数据要传输了，要求释放连接。

我们看回这里的报文值：`0x018`，即二进制的`0000 0001 1000`，其中第11、12位为1，即ACK和PSH的值为1。

第14、15个字节表示窗口大小，`0x0804`即`2052`。`[Calculated window size: 131328]`表示放大后的窗口大小为`131328`（2052*64），由于只有2个字节用来表示窗口大小，所以最大表示的窗口大小为64K，为了表示更大的窗口，这里使用了放大倍数：`[Window size scaling factor: 64]`。这个放大倍数是在哪里来的呢？其实是在建立TCP连接时的3次握手中通过TCP首部的选项字段指定的。

第16、17个字节表示校验和，此处值为：`0x01aa`。第18、19个字节表示紧急指针，此处值为：`0x0000`。

再后面的字节表示选项字段，最多40个字节，此处为12个字节：`0x01 01 08 0a 07 6b 77 88 e8 4b d4 30`。

# HTTP

![](/images/wireshark/Wireshark-HTTP.png)

![](/images/wireshark/Wireshark-HTTP-2.png)

到了HTTP这部分看起来就比较轻松了，本文不再赘述。

[comment]: <> (后续：Wireshark系列 2 —— TCP握手与挥手分析)
[comment]: <> (后续：Wireshark系列 3 —— HTTPS传输分析)
[comment]: <> (后续：Wireshark系列 4 —— MySQL传输分析)
[comment]: <> (后续：Wireshark系列 5 —— Redis传输分析)