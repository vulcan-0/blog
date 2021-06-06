---
title: Wireshark系列（4） —— HTTPS传输分析
date: 2021-01-20 20:44:28
tags:
    - 网络
    - Wireshark
categories: 网络
---

本文通过抓取HTTPS包，结合TLS协议内容，对HTTPS传输过程进行分析。

## 抓取HTTPS数据包

本节展示的是Mac系统下，使用Wireshark抓取HTTPS数据包的方法。

1. 启动Chrome，并将HTTPS握手过程产生的数据写到文件`/tmp/ssl-key.log`中。

``` shell
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --user-data-dir=/tmp/chrome --ssl-key-log-file=/tmp/ssl-key.log
```

2. 执行命令`tail -f /tmp/ssl-key.log`，如果能看到相应的内容，说明握手数据能成功写入文件。

![](/images/wireshark/4/Wireshark-HTTPS-capture1.png)

3. 选择`Wireshark` -> `Preferences...` -> `Protocols` -> `TLS`，在`(Pre)-Master-Secret log filename`下面输入上面的文件目录`/tmp/ssl-key.log`，点击`OK`，保存设置。

![](/images/wireshark/4/Wireshark-HTTPS-capture2.png)

经过上面步骤的设置，就可以成功抓取到HTTPS的内容了：

![](/images/wireshark/4/Wireshark-HTTPS-capture3.png)

## TLS握手过程分析

### TLS协议

### TLS握手过程