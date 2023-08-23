---
title: Wireshark使用不完全指南
date: 2021-09-17 16:36:51
tags: ["Wireshark", "TCP", "HTTP", "HTTPS"]
cover: banner.jpeg
categories: 学习
---

## 前言
最近在网络协议方面的书，书中介绍的学习网络协议最重要的工具就是Wireshark，这是一个开源的工具，具有强大网络分析功能。不同于其他抓包工具只能抓取应用层数据包，Wireshark可以分析查看各个协议层的数据包，功能极其强大，最重要的该工具是免费的。但也正是由于功能强大灵活，所以使用起来也比较复杂，不仅需要具备一些网络协议的基础知识，其过滤器的写法也有一定的学习成本，可以说学习使用该软件过程主要就是要学习这些过滤器的写法。只有熟悉了过滤器的写法，才能从Wireshark抓取的海量网络数据中找到自己想要的包。下面对Wireshark的基本功能和过滤器写法做一个基本介绍作为备忘，读完本文对Wireshark的掌握基本也能够应付日常抓包分析的需要。
## 基本界面和功能
以Mac OS为例，打开Wireshark的界面如下：
![Wireshark-Welcome.png](Wireshark-Welcome.png)

Wireshark不像其他应用层的HTTP抓包工具需要在本机启动一个代理服务器，他是直接工作在计算机的网卡之上的，直接对计算机网卡的网络数据进行分析，所以在启动软件欢迎界面，我们需要选择要监听的网卡，根据不同的计算机，网卡的数量是不同的，一般en0是计算机上的主网卡，我们可以选择这个，网卡列表的的上方和菜单栏的下方各有一个输入框，该输入框可以填写抓取过滤器（Capture Filter）。Wireshark主要包含两种过滤：Capture Filer和Display Filter。前者主要通过表达式决定要抓取哪些符合条件的数据包，不满足条件的网络数据包忽略掉。例如`host www.baidu.com`，只抓取`www.baidu.com`这个域名下的包，其他的包不抓取。当然它也可以置空。Display Filter则是只展示那些满足表达式要求的包，不满足条件的包则不会展示在列表中，主要负责从海量的数据中获取自己想要查看的记录。例如：`tcp.port == 80`（注意端口号port是传输层的概念）。选择了网卡之后，可以点击右上的蓝色鲨鱼鳍图标开始抓取。**一个好的实践时抓到自己想的数据包后，就可以点击右边的红色矩形按钮暂停抓取。**打开后界面如下
![Wireshark-capture.png](Wireshark-capture.png)
该页面主要分为三部分，上面的列表展示抓取结果，不同的颜色代表不同类型的数据，NO.列左侧的方括号之内的数据属于同一个网络对话，向右的箭头表示发出的请求，向左的箭头表示收到的响应。比如一次HTTP请求，包含一个应用层的请求，传输层的TCP三次握手，和一次应用层的响应。列表的上边的输入框用来填写Display Filter，后文详细讲述其写法。
中间的部分根据列表中选择数据类型的不同，展示也会略有不同，主要是能够查看不同协议层的数据头：
- Frame 主要是物理层的原始数据帧，调试蓝牙等硬件需要关心这一层的数据。
- Ethernet 主要是数据链路层数据头。
- Internet Protocol Version 4 网络层数据头，图中是IPv4。
- Transmission Control Protocol 传输控制层的TCP协议，查看三次握手的过程主要关注TCP协议数据头。
- Hypertext Transfer Protocol 应用层的HTTP协议。
以上层各种数据根据选择的数据的不同，展示的结果也不完全一样，比如如果选择的是一个TCP数据包，下方不会展示应用层数据，如果选择的是UDP数据包，则传输层展示的会是UDP数据，网络层也有可能是IPv6协议等等，如果选择是的HTTPS请求，在应用层和传输层之间还有一个Transport Layer Security(TSL)层。

最下面一部分则是所选数据数据包的16进制原始数据，单击数据，左侧会高亮显示所选数据原始数据对应的明文数据。在原始16进制数据上单击右键在菜单中选择...as bits，可以以二进制的形式查看数据。最下方的状态栏则是对抓取结果的汇总。
前面说过，过滤器写法是使用Wireshark难点之一，下面他们总结
## Capture Filter
抓取过滤器主要设置要抓取哪些包，限制抓取包的数量，可以在在欢迎界面设置，也可以在开始抓取之后通过菜单栏的<u>**Capture**</u> > <u>**Options**</u>修改，其语法相对比较简答，表达式基本形式为：
```
Field [Value]
```
字段主要来自于TCP/IP协议族，可用的字段可以[参考这里](http://www.tcpdump.org/manpages/pcap-filter.7.html)，值可以省略，多个表达式可以使用逻辑连接符and、or，或者not。例如：
```
host 172.18.5.4
```
上面的过滤器含义是抓取源IP地址或者目标IP地址为172.18.5.4的通信数据
```
net 192.168.0.0/24
```
抓取目标IP地址和源IP地址范围为192.168.0.0/24的数据。

```
src net 192.168.0.0/24
```
```
dst net 192.168.0.0/24
```
也可以只抓取源IP或者目标IP地址的数据。


```
tcp port 80 and host 10.0.0.5
```
抓取TCP端口号为80的，并且目标主机或源主机地址为10.0.0.5的数据。

```
tcp port 23 and not src host 10.0.0.5
```
上面的过滤器含义是抓取23端口的，并且源主机地址不是10.0.0.5的数据。
```
ip
```
只抓取IPv4的数据包，ip6抓取IPv6的数据。

```
tcp portrange 1501-1549
```
抓取端口号范围为1501-1549的包。

```
host www.example.com and not (port 80 or port 25)
```
抓取特定域名住的的非HTTP和非SMTP通信数据。
## Display Filter
相比抓取过滤器，显示过滤器更为常用，其表达式基本形式为：
```
protocol.field operator value
```
protocol为网路协议名称，field是协议下的某个字段。Wireshark支持的网络协议众多，其支持的所有协议以及某个协议下的字段可以在<u>**View**</u> > <u>**Internals**</u> > <u>**Supported Protocols**</u>下查看。

![Wireshark-Supported-Protocols.png](Wireshark-Supported-Protocols.png)

### 操作符
Opsrator指的的操作符，主要是比较操作符，操作符既可以使用编程语言里常见的比较符，也可以使用英文缩写，如`tcp.port == 80`或`tcp.port eq 80`，所有支持的操作符如下：

| 简写 |符号  |描述  |例子  |
| --- | --- | --- | --- |
|eq  |==  |相等  |`ip.src==10.0.0.5`  |
|ne  |!=  |不等  |`ip.src!=10.0.0.5`  |
|gt  |>  |大于  |`frame.len > 10`  |
|lt  |<  |小于  |`frame.len < 128`  |
|ge  |>=  |大于等于|`frame.len ge 0x100`  |
|le  |<=  |小于等于|`frame.len <= 0x20`  |
|contains|  |包含  |`sip.To contains "a1762"`|
|matches|~|字符串类型匹配正则表达式|`http.host matches "acme\\\\.(org\|com\|net)"` |
|bitwise_and|&|按位与|`tcp.flags & 0x02`|

### 字段类型
显示过滤器字段是有具体类型的，所有的类型如下：

|类型| 描述 | 例子 |
| --- | --- | --- |
|Unsign integer|无符号整型，可以是十进制，十六进制，八进制的数字|`ip.len le 1500`  |
|Sign integer| 同上 |  |
|Boolean|布尔，1代表true，0代表false，如TCP三次握手时的头信息的符号位SYN，ACK，FIN|`tcp.flags.syn`过滤TCP头的SYN位为0或1，`tcp.flags.syn == 1`过滤改符号为为1的数据包|
| Ethernet address|以太网地址，通过`:`， `.`， `-`分隔的六字节地址  |`eth.dst == ff:ff:ff:ff:ff:ff` `eth.dst == ff-ff-ff-ff-ff-ff` `eth.dst == ffff.ffff.ffff`  |
| IPv4 address | IPv4地址 |`ip.addr == 192.168.0.1` `ip.addr == 129.111.0.0/16`|
| IPv6 address | IPv6地址 |`ipv6.addr == ::1`|
| Text string | 文本字符串 |`http.request.uri == "https://www.wireshark.org/"`，可以使用ASSIC码十六进制（\xhh）或者八进制(\ddd)数字，如`dns.qry.name contains "www.\x77\x69\x72\x65\x73\x68\x61\x72\x6b.org"`，也支持正则表达式:`http.user_agent matches r"\(X11;"` |

### 逻辑连接符

还可以用逻辑连接符连接多个基本基本表达式

| 简写 |操作符  |描述  |例子  |
| --- | --- | --- | --- |
|and  | && |逻辑与  | `ip.src==10.0.0.5 and tcp.flags.fin` |
|or  | \|\| |逻辑或  |`ip.src==10.0.0.5 or ip.src==192.1.1.1`  |
|xor  | ^^ | 异或运算符，其值为真仅当两个运算元中恰有一个的值为真，而另外一个的值为非真 |`tr.dst[0:3] == 0.6.29 xor tr.src[0:3] == 0.6.29`  |
|not  | ! | 逻辑非 |`not llc`  |
|[...]  |  | 切片操作 |  |
|in  |  |成员操作符  | `http.request.method in {"HEAD" "GET"}` |

切片操作允许选择字段的子序列，例子如下：
```
eth.src[0:3] == 00:00:83
// src字段的开头三个值0位开始位置，3是长度

eth.src[1-2] == 00:83
// src字段的从1为开始到第2位

eth.src[:4] == 00:00:83:00
// 返回从开始位置，长度为4的字串

eth.src[4:] == 20:20
// 返回从第4位到结束的字串

eth.src[2] == 83
// 第2位的字串

eth.src[0:3,1-2,:4,4:,2] ==
00:00:83:00:83:00:00:83:00:20:20:83
以上切片表达式用","将所有字串组合在一起
```
成员操作符用于检测给定的字段是不是出现在给定的集合当中，如：
```
tcp.port in {80 443 8080}
```
等价于以下内容：
```
tcp.port == 80 || tcp.port == 443 || tcp.port == 8080
```

还可以使用范围操作符：
```
tcp.port in {443 4430..4434}
```
等价于
```
tcp.port == 443 || (tcp.port >= 4430 && tcp.port <= 4434)
```
### 函数
显示过滤器还支持函数：

| 函数 |描述  |例子  |
| --- | --- | --- |
|`upper`|将字段转换为大写  | `lower(http.server) contains "NGINX"` |
|`lower`|将字段转换为小写  |`lower(http.server) contains "apache"`  |
|`len`|返回给定字段的字节长度  |`len(http.request.uri) > 100`  |
|`count`|返回某个字段在数据帧中出现的次数  |`count(ip.addr) > 2`|
|`string`|将非字符串类型字段转换为字符串  |`string(frame.number) matches "[13579]$"`|

### 注意事项
在使用显示过滤器的时候，首先要注意的是，字段和所属的协议必须要对应，比如初学者常见错误：`http.port == 80`或`http.addr == 1.2.3.4`，这样的过滤器写法是错误的，因为端口号是应用层如TCP协议的字段，类似的addr也属于网络层的字段，对于非法的过滤器，Wireshark会给出提示。
另外一个错误是!=操作符的使用，表达式`ip.addr == 1.2.3.4`可能会匹配到IP地址为1.2.3.4，因为ip.addr包括网络层的源IP地址和目标IP地址两个字段，只要有一个字段不等于1.2.3.4，该数据即满足条件。类似的还有`eth.addr`，`ip.addr`，`tcp.port`和`udp.port`。

以上是对显示过滤器的总结，看起来很麻烦，但是实际使用时会发现Wireshark提供很多人性化的设计，比如显示过滤器输入框的左侧书签按钮内置了很多常用过滤器，最右边的向下的箭头包含了过滤器的历史记录。非法的过滤器表达式，输入框变成红色进行提示，所以在熟悉网络协议的基础上，完全可以凭感觉写出正确的过滤器表达式。

## 个性化定制
Wireshark的UI是支持定制的，主要是个性化的设置抓取结果列表，默认的表格对于调试可能不太方便。例如默认Time列显示的时间是从点击开始抓取的时间。可以在菜单栏<u>**View**</u> > <u>**Time Display Format**</u>中选择合适的时间格式，建议选择Time of Day。默认的列也可以修改，以添加Host，Source Port和Destination Port列为例：
![Wireshark-Column.png](Wireshark-Column.png)
在菜单栏里选择<u>**Wireshark**</u> > <u>**Preference**</u> > <u>**Appearance**</u> > <u>**Columns**</u><u></u>。点击加号，Title输入Source Port，Type选择如图所示。同样的方式添加Destination Port，Type字段如图所示。
接下来用另外一种简便的方式演示添加Host字段，先点击绿色的鲨鱼鳍按钮，清空当前的抓取结果，然后打开命令行工具打开输入：
```
curl http://wttr.in/Shanghai?0
```
然后点击红色的方块按钮暂停抓取，在显示过滤器里输入
```
http.host == wttr.in
```
来直接定位到我们刚才发送的http请求，点击查看应用层的HTTP数据，右键点击`Host: wttr.in\r\n`所在行，在弹出的下拉菜单中选择<u>**Apply as Column**</u>，可以看到表格添加了新的Host列，然后可以根据喜好，拖动列的位置或调整宽度。还可以在在表头的Source Port列上单击右键，在弹出的菜单中选择**Align Left**，使内容左对齐。
![Wireshark-Host-Column.png](Wireshark-Host-Column.png)

虽然强烈不建议修改，但是Wireshark也支持自定义抓取结果的颜色，可更改现有的颜色或添加新的颜色定义，具体方式在菜单栏选择：<u>**View**</u> > <u>**Coloring Rules...**</u>　
![Wireshark-Color.png](Wireshark-Color.png)
这里也可以查看默认颜色所代表的含义。或者点击加号添加新的颜色定义。
## 抓取HTTPS包
你可能已经发现在浏览器里访问的很多网站，Wireshark是抓取不到对应的HTTP请求的，这是因为目前很多主流的网站都已经采用HTTPS了，Wireshark默认是无法抓取这些数据的，接下来以Mac OS和Firefox演示如何抓取浏览器中的HTTPS请求。其他操作系统和浏览器的原理相同。首先需要创建环境变量并使其生效，在命令行执行：
```
#zsh
echo "\nexport SSLKEYLOGFILE=~/tls/tlskeylog.log" >> ~/.zshrc && source ~/.zshrc
```
如果是使用的是其他Shell，也可以执行：
```
#bash
echo "\nexport SSLKEYLOGFILE=~/tls/tlskeylog.log" >> ~/.bash_profile && . ~/.bash_profile
```
接着创建~/tls/tlskeylog.log文件：
```
mkdir ~/tls && touch ~/tls/tlskeylog.log
```
通过命令行启动浏览器，启动之前保证浏览器没有正在运行，否则设置不生效：
```
open /Applications/Firefox.app
```
尝试在浏览器里打开https://www.baidu.com，然后打开在Wireshark选择<u>**Wireshark**</u> > <u>**Preference**</u>，在弹出的对话框中选择Protocols下的TLS，选择刚才创建的文件后点击OK：
![Wireshark-TLS.png](Wireshark-TLS.png)

刷新浏览器，并在Wireshark里输入显示过滤器，即可看到https：
```
http.host == "www.baidu.com"
```
其原理是通过设置特殊的环境变量，使浏览器和访问的服务器在进行HTTPS握手的过程中产生的随机数和最后用于对称加密的密钥保存在外部文件中，然后Wireshark读取该文件，对抓取到的加密数据进行解密。如果打开tlskeylog.log文件应该能够看到`CLIENT_RANDOM`、`CLIENT_HANDSHAKE_TRAFFIC_SECRET`和`SERVER_HANDSHAKE_TRAFFIC_SECRET`字段。对HTTPS握手过程不明白的可以看[这篇文章](https://mp.weixin.qq.com/s/CAcYxsNe2g6Wk4rF5D9d7Q)，即能够理解该设置的原理。
## 总结
Wireshark作为一个强大的网络抓包分析工具，主要功能是对网络数据包进行分析，并且可以对抓到的网络数据包导出到文件。

## 参考链接
- https://www.wireshark.org/docs/wsug_html_chunked/
- https://gitlab.com/wireshark/wireshark
- https://unit42.paloaltonetworks.com/unit42-customizing-wireshark-changing-column-display/
- https://imququ.com/post/http2-traffic-in-wireshark.html
