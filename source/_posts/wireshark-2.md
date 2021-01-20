---
title: Wireshark系列（2） —— TCP stream
date: 2021-01-01 21:39:56
tags:
    - 网络
    - Wireshark
categories: 网络
---

本文将透过一个HTTP请求，结合TCP协议的内容，对其中涉及到的TCP数据包进行详细分析。

![](/images/wireshark/2/Wireshark-TCP-stream.png)

## 建立连接

TCP协议的目的，是为了保证通讯双方的数据能够准确连续地流动，可以将TCP通道比作一根水管，有了这根水管，两端的水就能够自由地流动了。为了让一个设备能连接多根“水管”，并且不同“水管”的水流互不干扰，TCP协议利用socket数据结构实现不同设备间的连接，socket封装了两端的IP和端口。通讯双方一般分为客户端和服务器，建立连接前，服务器需要监听指定端口。客户端发起连接前需要先建立好TCB，服务器收到连接请求，会被动建立TCB。我们可以把TCB看成一个信封，它封装了socket信息以及装载数据的缓冲区。[参考博文](https://cloud.tencent.com/developer/article/1474773)

![](/images/wireshark/2/Wireshark-TCP-handshake.png)

### 三次握手

服务器开始监听端口，状态变为LISTEN。客户端发起连接请求时，向服务器发送窗口大小、最大分段大小、顺序号等信息，并将自身的状态置为“SYN-SENT”，表示已发送同步请求。服务器收到连接请求，会将自身的窗口大小、最大分段大小、顺序号等，连同客户端的顺序号+1作为确认号发送给客户端，并将自身的状态置为“SYN-RECEIVED”，表示已收到同步请求。客户端收到服务器返回的信息，则将服务器的顺序号+1作为确认号发送给服务器，并将自身状态置为“ESTABLISHED”，表示已连接。服务器收到客户端的确认信息，则将自身状态置为“ESTABLISHED”，表示已连接。至此，三次握手完成，TCP连接建立成功。

![](/images/wireshark/2/Wireshark-TCP-handshake-2.png)

上图展示的3个包，就是三次握手过程中产生的数据包，我们接下来分别看下这几个包的数据。

#### SYN

![](/images/wireshark/2/Wireshark-TCP-handshake-package1.png)

从172.20.77.71:50410发给168.62.224.13:80，SYN标志位为1，172.20.77.71是我本地的IP地址，168.62.224.13是一个远端服务器的地址，即此包为本地客户端向远端服务器发起的一个SYN包。

##### 随机序号

```javascript
Sequence number: 0    (relative sequence number)
Sequence number (raw): 2196773886
Acknowledgment number: 0
Acknowledgment number (raw): 0
```

这里有一个相对的顺序号、确认号，和一个原始的顺序号、确认号。我们考虑这样一种情况，假设三次握手时，客户端发送的SYN包的顺序号是0，但此时网络拥堵，导致连接没有建立成功。客户端只好再次发起握手，尝试连接，如果这时候网络通畅了，服务器收到了客户端上次发送的包，并以此为依据建立连接，则会导致连接出现异常。假设两次连接请求发送的数据是不一样的，建立的连接就会以旧的数据为准。因此，为避免这种情况的发生，建立TCP连接时的初始顺序号必须是随机的，客户端在收到服务器的ACK包时，就可以根据确认号判读是否应该继续建立连接。随机的序号不好辨认，所以Wireshark给我们提供了一个相对的序号，方便我们追踪请求。为了方便追踪，我们关注的也是相对序号，此包的相对顺序号为0，相对确认号也为0。（如无特别说明，后面说的顺序号为相对顺序号，确认号为相对确认号。）

在大部分情况下，发送一个数据包的顺序号=上一个数据包的顺序号+上一个数据包的数据长度，返回一个数据包的确认号=收到的数据包的顺序号+收到的数据包的数据长度。在应答SYN包|FIN包时是比较特殊的，对于一个应答SYN包|FIN包的ACK包，它的确认号=SYN包|FIN包的顺序号+1，收到该应答包的一方，会以该包的确认号作为下一个发送的数据包的顺序号。

##### 窗口大小

窗口大小指的是由接收端决定的滑动窗口的大小，滑动窗口主要解决的问题是，发送端和接收端速度不匹配的问题，窗口大小受到应用、系统、硬件的限制。最初的时候，窗口大小最大为2的16次方-1，即65535字节，随着带宽的不断提高，65535字节已经不够用了，因而引入了`Window Scale`，通过`Window Scale`放大窗口的大小，实际窗口大小=`Window size`左移`Window Scale`位。这里的`Window Scale`为6，左移6位相当于放大2的6次方倍，即64倍。`Window Scale`指的是接收端的窗口放大倍数，即接收端的窗口大小值，最终为`Window size`按照`Window Scale`放大后的大小。客户端和服务器在三次握手中的SYN和SYN+ACK包指定`Window Scale`，指定之后的数据包开始沿用该值。

##### 分段大小

MSS（Maximum segment size），即分段大小，此处为1460。我们回忆一下上一篇谈到的，以太网协议规定了以太网帧的数据段最大大小为1500字节，所以TCP包分段大小一般不大于1500，否则会需要用到IP包的分片功能，这被认为是低效的。另外，IP头大小为20字节，TCP头最小大小为20字节，所以此处的分段大小设置为：1500-20-20=1460字节，分段大小指的是TCP数据段的大小。

![](/images/wireshark/1/Wireshark-Ethernet-3.png)

> 在SYN包的Options中出现了一个SACK（Selective Acknowledgment，选择性确认），SACK是什么意思呢？可以参考这篇博文：[TCP重点系列之sack介绍](https://allen-kevin.github.io/2017/03/01/TCP%E9%87%8D%E7%82%B9%E7%B3%BB%E5%88%97%E4%B9%8Bsack%E4%BB%8B%E7%BB%8D/)；

#### SYN+ACK

![](/images/wireshark/2/Wireshark-TCP-handshake-package2.png)

可以看到，此包为SNY+ACK包，顺序号为0，确认号为1（SYN包的顺序号+1）；窗口大小为8192，窗口放大倍数为256；分段大小为1440，各服务器会根据自身所处的网络环境对分段大小进行调整，一般不超过1460。

#### ACK

![](/images/wireshark/2/Wireshark-TCP-handshake-package3.png)

ACK包以服务器的SYN+ACK包的确认号为顺序号，目前为1，确认号为1（SYN+ACK包的顺序号+1）；窗口大小此时发生了变化，为2052，乘以倍数64后为131328。

> 在此，提一个问题：三次握手为什么是充分必要的？首先，TCP在建立连接前，必须将自身的信息（如窗口大小、最大分段大小等）发给对方，否则双方不知道能够向对方使用什么策略发送数据（比如一次能发多少数据等）。另外，为何必须收到对方的确认才认为连接是成功建立了呢？要回答这个问题，就要提到随机序号了，我们上面已经说过SYN包的初始顺序号是随机的，这个随机序号在握手确认时至关重要。假设存在这样一种情况：TCP在建立连接的时候，刚好发生网络拥堵，这个时候客户端发送了两个SYN包，如果两个SYN包的数据是不一样的，而服务器收到了第一个SYN包就确定了连接的建立，这显然是不合理的。所以，客户端需要接收服务器的ACK，如果ACK的序号跟预期是一样的，则继续往下建立连接，如果不一样，则发送RST包重置该连接。反过来，服务器向客户端发送初始数据的逻辑也一样。走完这些流程，刚好至少需要三次交互，由此可知，三次握手是充分必要的。

将此TCP stream的三次握手过程绘制成图，如下：

![](/images/wireshark/2/Wireshark-TCP-handshake2.png)

## 传输数据

### 滑动窗口

我们简单回顾下滑动窗口的算法：

![](/images/wireshark/2/Wireshark-TCP-sliding-window1.png)

发送端缓存的数据包含4部分：已发送已确认的、已发送未确认的、未发送待发送的、未发送不准备发送的，已发送未确认的+未发送待发送的=发送窗口大小。

![](/images/wireshark/2/Wireshark-TCP-sliding-window2.png)

接收端存在3种状态：已接收已确认的、未接收准备接收的、未接收不准备接收的，未接收准备接收的=接收窗口大小。

对照以上几种状态，我们就大致知道滑动窗口的基本策略了：发送端在发送窗口未满的时候可以不断发送数据，在收到确认的时候右移窗口；接收端在收到数据并发送确认之后，右移窗口。

![](/images/wireshark/2/Wireshark-TCP-stream2.png)

![](/images/wireshark/2/Wireshark-TCP-stream3.png)

对照上面两张图，我们可以发现传输过程中，本地的接收窗口在不断发生变动，服务器的接收窗口一直没有变化，其中最小的窗口值为123584字节，大约为120KB。我们再看到编号为4492的请求，它把4482、4483...4492这8个TCP的数据拼装在了一起，组成了HTTP的返回信息，可以看到它的总大小为8880字节（注意：这是8个TCP包合并起来的数据），大约为8KB，再加上前面建立连接和后面断开连接的数据，一共也不超过9KB，也就是总长度最大不会超过最小的窗口大小，所以在此TCP stream的传输过程中，其实是没有发生窗口移动的，发送端一直都处于窗口未满的状态。

### HTTP数据流

为了便于观察，把此HTTP请求包含的几个TCP数据包的传输过程绘制成下图：

![](/images/wireshark/2/Wireshark-TCP-HTTP-stream.png)

在Wireshark中输入`http`将HTTP请求包和返回包过滤出来：

![](/images/wireshark/2/Wireshark-TCP-HTTP-stream2.png)

HTTP请求和返回只有一来一回的两个包，为什么上图的TCP包会有这么多个呢？我们仔细观察，可以发现，帧序号4465的HTTP请求包就是序号为4465的TCP包，帧序号4492的HTTP返回包就是服务器最后一个返回的序号为4492的TCP包。其实在一个HTTP请求包或返回包中，就可能包含多个TCP包。此HTTP请求的数据只有105字节，一个TCP包就把数据传输完了，因此也不必传输多个，Wireshark也直接将此包的协议显示为HTTP。服务器接收到HTTP请求的数据后，会先返回ACK，再开始把HTTP返回数据传递给客户端。我们之前已经知道，此HTTP返回的数据总共是8880字节，由于此TCP通道一次只能传输1千多个字节，所以分了多次进行传输。其中，服务器每传完2~3个包就发一个PSH包，客户端收到PSH包会马上返回ACK给服务器，经过13个TCP包的交互，返回的HTTP数据终于传完。其中的4492，为什么显示为HTTP包，其他的返回包均显示为TCP包呢？这其实是Wireshark的一个处理，Wireshark在检测到HTTP返回数据传输完成，就把最后一个数据返回包显示为HTTP包。

## 断开连接

### 四次挥手

客户端发送FIN包，状态变为FIN-WAIT-1；待收到服务器ACK包时，状态变为FIN-WAIT-2；收到服务器发过来的FIN包时，状态变为TIME-WAIT；再等待2MSL之后，连接状态最终变为CLOSED。服务器收到FIN包时，状态变为CLOST-WAIT；等到向客户端发起FIN包时，状态变为LASK-ACK；收到客户端ACK包时，连接状态最终变为CLOSED。

![](/images/wireshark/2/Wireshark-TCP-fin-handshake.png)

当然，也可能是客户端和服务器同时发送FIN包，此时双方的状态变为FIN-WAIT-1；当收到对方的FIN包并发送ACK包时，状态变为CLOSING；当收到对方的ACK包时，状态变为TIME-WAIT；等待2MSL之后，连接状态最终变为CLOSED。

![](/images/wireshark/2/Wireshark-TCP-fin-handshake2.png)

将此TCP stream的四次挥手过程绘制成图，如下：

![](/images/wireshark/2/Wireshark-TCP-fin-handshake3.png)

其中，多出一个帧序号为4497的ACK包，这其实是本地向服务器发起的一个变更接收窗口大小的包。

![](/images/wireshark/2/Wireshark-TCP-fin-handshake4.png)

> 四次挥手为什么是必要的呢？发送FIN包一方的目的在于告诉对方，将不再发送数据包给对方，但对方还是可以向自己发送数据包的，因此断开连接时，双方都要告诉对方，将不再发送数据包给对方。而确认包其实已经成为了每一个发出去的包的标配，否则不知道对方是否真的收到了，只有确定每次发出去的数据包，对方都能收到，连接才是可靠的。能否将四次挥手合并为三次呢？其实不行，因为收到FIN包的一方可能还没准备好断开连接，所以需要给它一定的时间，准备好之后，再向发起方也发送一个FIN包。能否等到自身准备好之后，再把FIN包和ACK包一起发回给发起方呢？也是不行的，如果自身准备耗时过长，将导致发起方不断发起FIN包，这显然也是不可取的。

[comment]: <> (2MSL？)

> 说明：本系列使用的Wireshark版本为Mac系统下的3.2.2版本。