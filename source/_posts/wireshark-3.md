---
title: Wireshark系列（3） —— DNS
date: 2021-01-18 18:47:55
tags:
    - 网络
    - Wireshark
categories: 网络
---

本文通过抓取DNS包，结合DNS协议，对域名解析过程进行分析。

## UDP

DNS是基于TCP和UDP的一个应用层协议，在更较多情况下用的是UDP，因此在分析DNS包之前，我们先回顾一下UDP的内容。

![](/images/wireshark/3/Wireshark-DNS-UDP.png)

![](/images/wireshark/3/Wireshark-DNS-UDP2.png)

UDP（User Datagram Protocol，用户数据报协议）是一种简单的面向数据报的通讯协议，它为应用程序提供了一种无须建立连接就可以发送数据包的方法。它的协议内容比较简单，包含来源连接端口、目的连接端口、报文长度、校验和几个部分。因为UDP不需要应答，所以它的来源连接端口是可选的，如果不用，则置为0。在IPv4中，校验和也是可选的，但实际应用中，一般都会用上。报文长度字段占16位，因而UDP数据的总长度不能超过65507字节（65535 − 8字节UDP报头 − 20字节IP头部）。上面展示的UDP包的源连接端口为`30323`，目的连接端口为`53`，数据长度为`44`，校验和为`0x3344`。

## DNS

DNS协议定义了统一的消息格式，它的请求报文包含了请求的问题，返回报文包含了请求的问题和对问题的回答。

![](/images/wireshark/3/Wireshark-DNS-DNS0.png)

DNS的头部内容包含2字节的会话标识、2字节的标志位、2字节的问题计数、2字节的回答资源记录数、2字节的权威名称服务器计数和2字节的附加资源记录数。

![](/images/wireshark/3/Wireshark-DNS-DNS1.png)

其中，各标志位的含义如下：

![](/images/wireshark/3/Wireshark-DNS-DNS2.png)

DNS记录类型的含义如下：

![](/images/wireshark/3/Wireshark-DNS-DNS3.png)

下面的报文表示查询报文，期望递归，问题数为1，请求该域名指向的IP地址。查询的目标地址为`10.254.254.254`，也就是本地网络的DNS服务器的地址。（Class IN，表示查询的是互联网地址。）

![](/images/wireshark/3/Wireshark-DNS-DNS4.png)

应答报文如下：

![](/images/wireshark/3/Wireshark-DNS-DNS5.png)

查询域名`zhuanlan.zhihu.com`最终定位到了域名`j1hqc7ee.shced.d0.tdnsv5.com`，域名`j1hqc7ee.shced.d0.tdnsv5.com`指向了11个IP地址。它的授权服务器域名为`d0.tdnsv5.com`，它的映射信息记录在域名为`ns1.d0.tdnsv5.com`的服务器中，其中，`ns1.d0.tdnsv5.com`指向了7个具体的IP地址。

[comment]: <> (单播、组播、任播；DNSmasq)