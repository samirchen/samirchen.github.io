---
layout: post
title: Linux环境下网络性能测试
description: 介绍一些Linux环境下网络性能测试常用的工具和具体使用。
category: blog
---

##网络性能测试的几项重要指标

1、可用性

测试网络性能的第一步是确定网络是否正常工作，最简单的方法就是使用`ping`命令，通过向远端的机器发送ICMP请求，并等待接收ICMP回应，来判断远端的机器是否连通，网络是否正常工作。

2、响应时间

`ping`命令的ICMP报文响应一次往返所花费时间就是响应时间，有很多因素会影响到响应时间，如网段的负荷，网络主机的负荷，广播风暴，工作不正常的网络设备等等。

3、网络利用率

网络利用率是指网络被使用的时间占总时间（即被使用的时间+空闲的时间）的比例。例如，Ethernet虽然是共享的，但同时却只能有一个报文在传输，因此在任一时刻，Ethernet或者是100%的利用率，或者是0%的利用率。计算一个网段的网络利用率相对比较容易，但是确定一个网络的利用率就比较复杂。因此，网络测试工具一般使用网络吞吐量和网络带宽容量来确定网络中两个节点之间的性能。

4、网络吞吐量

网络吞吐量是指在某个时刻，在网络中的两个节点之间，提供给网络应用的剩余带宽，通过网络吞吐量可以寻找出网络瓶颈。比如，即使`client`和`server`都被分别连接到各自的`100M`以太网卡上，但是如果这两个`100M`的以太网卡被`10M`的交换机连接起来，那么`10M`的交换机就是网络的瓶颈。

5、网络带宽容量

与网络吞吐量不同，网络带宽容量指的是在网络的两个节点之间的最大可用带宽，这是由组成网络的设备能力所决定的。

##使用iperf进行测试

###iperf介绍

[iperf][]是一个TCP/IP和UDP/IP性能测试工具，能够提供网络吞吐率信息，以及震动、丢包率、最大组和最大传输单元大小等统计信息，可以由这些信息来分析网络的通信性能、定位网络瓶颈。

[iperf][]以client/server方式工作，服务器端和客户端都使用同一程序`iperf`，服务器端使用`-s`选项，而客户端则使用`-c`选项。在 client与server之间，首先建立一个控制连接，传递有关测试配置的信息，以及测试的结果；在控制连接建立并传递了测试配置信息以后，client与server之间会再建立一个测试连接，用来回传递着特殊的流量模式，以测试网络的性能。

###iperf获取与安装

下载地址：[http://iperf.sourceforge.net/](http://iperf.sourceforge.net/)

配置与安装：

	[root@rh tool]# tar -zxvf iperf-2.0.2.tar.gz
	[root@rh tool]# cd iperf-2.0.2
	[root@rh iperf-2.0.2]# ./configure --prefix=/usr/netperf
	[root@rh iperf-2.0.2]# make
	[root@rh iperf-2.0.2]# make install
	[root@rh iperf-2.0.2]# ls /usr/netperf

###iperf的使用

启动server端：

	[root@rh iperf-2.0.2]# cd /usr/iperf/bin
	[root@rh bin]# ./iperf -s

还可以设置这些参数：
<table>
<tbody>
<tr>
<th>服务器端选项</th>
<th>环境变量选项</th>
<th>说明</th>
</tr>
<tr>
<td>-s, --server</td>
<td>$IPERF_SERVER</td>
<td>在服务器端运行 iperf；</td>
</tr>
<tr>
<td>-D</td>
<td>.</td>
<td>以后台方式运行 iperf 服务模式；</td>
</tr>
<tr>
<td>-R</td>
<td>.</td>
<td>如果 iperf 正在运行，则将其终止运行；</td>
</tr>
</tbody>
</table>

启动client端：

用TCP的方式测试本机到`192.168.0.138`主机`（-c 192.168.0.138）`的网络性能，时长为`60`秒`（-t 60）`，缓冲区的大小为`8KB` `（-l 8k）`，每`10`秒`（-i 10）`打印一次测试结果。

	[root@rh iperf-2.0.2]# cd /usr/iperf/bin
	[root@rh bin]# ./iperf -c 192.168.0.138 -t 60 -l 8k -i 10
	------------------------------------------------------------
	Client connecting to 192.168.0.138, TCP port 5001
	TCP window size: 23.2 KByte (default)
	------------------------------------------------------------
	[  3] local 192.168.0.137 port 42812 connected with 192.168.0.138 port 5001
	[ ID] Interval       Transfer     Bandwidth
	[  3]  0.0-10.0 sec  1.64 GBytes  1.41 Gbits/sec
	[  3] 10.0-20.0 sec  5.26 GBytes  4.52 Gbits/sec
	[  3] 20.0-30.0 sec  5.26 GBytes  4.52 Gbits/sec
	[  3] 30.0-40.0 sec  5.27 GBytes  4.53 Gbits/sec
	[  3] 40.0-50.0 sec  5.26 GBytes  4.51 Gbits/sec
	[  3] 50.0-60.0 sec  5.26 GBytes  4.52 Gbits/sec
	[  3]  0.0-60.0 sec  27.9 GBytes  4.00 Gbits/sec

还可以设置这些参数：
<table>
<tbody>
<tr>
<th>客户端选项</th>
<th>环境变量选项</th>
<th>说明</th>
</tr>
<tr>
<td>-b, --bandwidth</td>
<td>$IPERF_BANDWIDTH</td>
<td>指定客户端通过 UDP 协议发送信息的带宽，默认值为 1Mbps；</td>
</tr>
</tbody>
</table>



##使用netperf进行测试

###netperf介绍

[netperf][]是一种网络性能测量工具，主要针对基于TCP或UDP的传输。[netperf][]根据应用的不同，可以进行不同模式的网络性能测试，即批量数据传输（bulk data transfer）模式和请求/应答（request/response）模式。[netperf][]反应的是一个系统能以多快的速度向另一个系统发送数据，以及另一个系统能以多快的速度接受数据。

[netperf][]是以client/server的方式工作，server端是netserver，用来侦听来自client 端的连接；client端是netperf，用来向server发起网络测试。在client与server之间，首先建立一个控制连接，传递有关测试配置的信息，以及测试结果；在控制连接建立并传递了测试配置信息后，client与server之间会再建立一个测试连接，用来来回传递着特殊的流量模式，以测试网络的性能。

[netperf][]可以模拟三种不同的TCP流量模式：

- 单个TCP连接，批量（bulk）传输大量数据；
- 单个TCP连接，client请求/server应答的交易（transaction）方式；
- 多个TCP连接，每个连接中一对请求/应答的交易方式

[netperf][]可以模拟两种UDP的流量模式：

- 从client到server的单向批量传输；
- 请求/应答的交易方式。

由于UDP传输的不可靠性，在使用 [netperf][]时要确保发送的缓冲区大小不大于接收缓冲区大小，否则数据会丢失，[netperf][]将给出错误的结果。因此，对于接收到分组的统计不一定准确，需要结合发送分组的统计综合得出结论。

###netperf获取与安装

下载地址：[http://www.netperf.org/netperf/](http://www.netperf.org/netperf/)

配置与安装：

	[root@rh tool]# tar -zxvf netperf-2.6.0.tar.gz
	[root@rh tool]# cd netperf-2.6.0
	[root@rh netperf-2.6.0]# ./configure --prefix /usr/netperf
	[root@rh netperf-2.6.0]# make
	[root@rh netperf-2.6.0]# make check
	[root@rh netperf-2.6.0]# make install
	[root@rh netperf-2.6.0]# ls /usr/netperf

###netperf使用

启动server端：

	[root@rh netperf-2.6.0]# cd /usr/netperf/bin
	[root@rh bin]# ./netserver

启动client端：

1）`TCP_STREAM`模式。进行TCP批量传输性能测试。这是[netperf][]缺省情况。

批量数据传输典型的例子有FTP和其他类似的网络应用（即一次传输整个文件）。根据使用传输协议的不同，批量数据传输又分为TCP批量传输和UDP批量传输。

用 `TCP 批量传输的方式` `（-t TCP_STREAM）`测试本机到 `192.168.0.138` 主机`（-H 192.168.0.138）`的网络性能，时长 `60` 秒`（-l 60）`，每次发送本地发送测试分组的大小为 `2048Bytes` `（-m 2048）`。

	[root@rh netperf-2.6.0]# cd /usr/netperf/bin
	[root@rh bin]# ./netperf -t TCP_STREAM -H 192.168.0.138 -l 60 -- -m 2048
	MIGRATED TCP STREAM TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.0.138 () port 0 AF_INET
	Recv   Send    Send
	Socket Socket  Message  Elapsed
	Size   Size    Size     Time     Throughput
	bytes  bytes   bytes    secs.    10^6bits/sec
	87380  16384   2048    60.00    5463.11

从上面[netperf][]的输出结果中，可以得到如下信息：

- 1）远端系统（即 server）使用大小为 87380 字节的 socket 接收缓冲；
- 2）本地系统（即 client）使用大小为 16384 字节的 socket 发送缓冲；
- 3）向远端系统发送的测试分组大小为 2048 字节，通过 -m 2048 设置；
- 4）测试经历的时间为 60 秒；
- 5）吞吐量的测试结果为 5463.11 Mbps；

在缺省的情况下，[netperf][]向远端系统发送的测试分组大小会设置为本地系统所使用的 socket 发送缓冲大小。

还可以设置这些局部命令行参数：

<table>
<tbody>
<tr>
<th>客户端选项</th>
<th>变量</th>
<th>说明</th>
</tr>
<tr>
<td>-s</td>
<td>$size</td>
<td>设置本地系统的 socket 发送与接收缓冲大小；</td>
</tr>
<tr>
<td>-S</td>
<td>$size</td>
<td>设置远端系统的 socket 发送与接收缓冲大小；</td>
</tr>
<tr>
<td>-m</td>
<td>$size</td>
<td>设置本地系统发送测试分组的大小；</td>
</tr>
<tr>
<td>-M</td>
<td>$size</td>
<td>设置远端系统接收测试分组的大小；</td>
</tr>
<tr>
<td>-D</td>
<td>.</td>
<td>对本地与远端系统的 socket 设置 TCP_NODELAY 选项；</td>
</tr>
</tbody>
</table>

通过修改以上的参数，并观察测试结果的变化，可以确定是什么因素影响了连接的吞吐量。

例如，如果怀疑路由器由于缺乏足够的缓冲区空间，使得转发大的分组时存在问题，就可以改变测试分组的大小`（-m）`，以观察吞吐量的变化。如果当测试分组由较大变为较小，而吞吐量出现较大的变化（比如吞吐量变大），说明网络中路由器确实存在缓冲区不足的问题。


2）`UDP_STREAM`，进行 UDP 批量传输性能测试。

需要注意的是此时测试分组的大小不能大于 socket 发送与接收的缓冲区大小，否则 netperf 会报错。

`UDP_STREAM` 方式使用与 `TCP_STREAM` 方式相同的局部命令行参数。

用 `UDP 批量传输的方式` `（-t UDP_STREAM）`测试本机到 `192.168.0.138` 主机`（-H 192.168.0.138）`的网络性能，时长 `60` 秒`（-l 60）`，每次发送本地发送测试分组的大小为 `2048Bytes` `（-m 2048）`。

	[root@rh netperf-2.6.0]# cd /usr/netperf/bin
	[root@rh bin]# ./netperf -t UDP_STREAM -H 192.168.0.138 -l 60 -- -m 2048
	MIGRATED UDP STREAM TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.0.138 () port 0 AF_INET
	Socket  Message  Elapsed      Messages
	Size    Size     Time         Okay Errors   Throughput
	bytes   bytes    secs            #      #   10^6bits/sec
	229376    2048   60.00     11159606      0    3047.29
	229376           60.00     11115165           3035.15

可以看到，最后结果有两行，第一行数据显示的是本地系统的发送统计，这里的吞吐量表示 netperf 向本地 socket 发送分组的能力。但是由于 UDP 是不可靠的传输协议，发送出去的分组数量不一定等于接收到的分组数量；第二行数据显示的就是远端系统接收的情况，这里看到接收到的数据小于发送的数据，即存在丢包的情况。远端系统的接收吞吐量也小于本地发送吞吐量。

3）`TCP_RR`，在一个 TCP 连接中进行多次 request 和 response 的交易过程的性能测试。

这种方式常出现在数据库应用中，数据库与客户端程序建立一个 TCP 连接后，就在这个连接中传递数据库的多次交易过程。

用 `TCP_RR` 的方式`（-t TCP_RR）`测试本机到 `192.168.0.138` 主机`（-H 192.168.0.138）`的网络性能，时长 `60` 秒`（-l 60）`，设置 request 分组大小为 `64Bytes`，response 分组大小 `1024Bytes`。

	[root@rh netperf-2.6.0]# cd /usr/netperf/bin
	[root@rh bin]# ./netperf -t TCP_RR -H 192.168.0.138 -l 60 -- -r 64,1024
	MIGRATED TCP REQUEST/RESPONSE TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.0.138 () port 0 AF_INET : first burst 0
	Local /Remote
	Socket Size   Request  Resp.   Elapsed  Trans.
	Send   Recv   Size     Size    Time     Rate
	bytes  Bytes  bytes    bytes   secs.    per sec
	16384  87380  64       1024    60.00    9194.92
	16384  87380

这里的输出结果有两行，第一行数据显示的是本地系统的信息，第二行数据显示的远端系统的信息。`Trans. Rate per sec` 是平均交易率。

通常增加 request/response 分组的大小会使得交易率下降。相对于实际的系统，这里的交易率的计算没有充分考虑到交易过程中的应用程序处理时延，因此结果会高于实际情况。

还可以设置这些参数：

<table>
<tbody>
<tr>
<th>客户端选项</th>
<th>变量</th>
<th>说明</th>
</tr>
<tr>
<td>-r</td>
<td>$req,$resp</td>
<td>设置 request 和 reponse 分组的大小；</td>
</tr>
<tr>
<td>-s</td>
<td>$size</td>
<td>设置本地系统的 socket 发送与接收缓冲大小；</td>
</tr>
<tr>
<td>-S</td>
<td>$size</td>
<td>设置远端系统的 socket 发送与接收缓冲大小；</td>
</tr>
<tr>
<td>-D</td>
<td>.</td>
<td>对本地与远端系统的 socket 设置 TCP_NODELAY 选项；</td>
</tr>
</tbody>
</table>


4）`TCP_CRR`，与 `TCP_RR` 的方式不同，`TCP_CRR` 为每次交易建立一个新的 TCP 连接。测试这种情况的网络性能。

这种方式最典型的应用是 HTTP，每次 HTTP 交易是在一条单独的 TCP 连接中进行的，这个过程需要不停地建立新的 TCP 连接，并在交易结束后结束 TCP 连接，交易率会受到较大影响。

用 `TCP_CRR` 的方式`（-t TCP_CRR）`测试本机到 `192.168.0.138` 主机`（-H 192.168.0.138）`的网络性能，时长 `60` 秒`（-l 60）`，设置 request 分组大小为 `64Bytes`，response 分组大小 `1024Bytes`。

	[root@rh netperf-2.6.0]# cd /usr/netperf/bin
	[root@rh bin]# ./netperf -t TCP_CRR -H 192.168.0.138 -l 60 -- -r 64,1024
	MIGRATED TCP Connect/Request/Response TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.0.138 () port 0 AF_INET
	Local /Remote
	Socket Size   Request  Resp.   Elapsed  Trans.
	Send   Recv   Size     Size    Time     Rate
	bytes  Bytes  bytes    bytes   secs.    per sec
	16384  87380  64       1024    60.00    2728.00
	16384  87380

这里的输出结果有两行，第一行数据显示的是本地系统的信息，第二行数据显示的远端系统的信息。`Trans. Rate per sec` 是平均交易率。

5）`UDP_RR`，使用 UDP 分组进行 request 和 response 交易过程的性能测试。

用 `UDP_RR` 的方式`（-t UDP_RR）`测试本机到 `192.168.0.138` 主机`（-H 192.168.0.138）`的网络性能，时长 `60` 秒`（-l 60）`，设置 request 分组大小为 `64Bytes`，response 分组大小 `1024Bytes`。

	[root@rh netperf-2.6.0]# cd /usr/netperf/bin
	[root@rh bin]# ./netperf -t UDP_RR -H 192.168.0.138 -l 60 -- -r 64,1024
	MIGRATED UDP REQUEST/RESPONSE TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.0.138 () port 0 AF_INET : first burst 0
	Local /Remote
	Socket Size   Request  Resp.   Elapsed  Trans.
	Send   Recv   Size     Size    Time     Rate
	bytes  Bytes  bytes    bytes   secs.    per sec
	229376 229376 64       1024    60.00    10635.03
	229376 229376

这里的输出结果有两行，第一行数据显示的是本地系统的信息，第二行数据显示的远端系统的信息。`Trans. Rate per sec` 是平均交易率。

`UDP_RR` 方式使用 UDP 分组进行 request/response 的交易过程。由于没哟 TCP 连接所带来的负担，所以交易率相对 `TCP_RR` 方式一般会有相应的提升。

但是如果出现了相反的结果，即交易率反而降低了，也不用太惊讶，因为这说明在网络中，路由器或其他网络设备对 UDP 采用了与 TCP 不同的缓冲区空间和处理技术。


##其他注意事项

测试的时候，常常由于防火墙的原因造成网络不能连接，这时候可以设置或关闭防火墙后再测试。关闭和启动防火墙命令：

	service iptables stop/start;

##后记

常用的网络性能测试工具除了 [iperf][]、[netperf][]，还有 [pathload][]、[pathrate][]、[DBS][]、[tcptrace][]等工具。

本文参考：[《网络管理必备工具软件精解（Linux版）》][]，作者：李波；杨红，人民邮电出版社。

[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{page.title}})
[iperf]: http://iperf.sourceforge.net/ "iperf"
[netperf]: http://www.netperf.org/netperf/ "netperf"
[pathload]: http://www.cc.gatech.edu/fac/constantinos.dovrolis/bw-est/pathload.html "pathload"
[pathrate]: http://www.cc.gatech.edu/~dovrolis/bw-est/pathrate.html "pathrate"
[DBS]: http://www.ai3.net/products/dbs/ "DBS"
[tcptrace]: http://www.tcptrace.org/ "tcptrace"
[《网络管理必备工具软件精解（Linux版）》]: http://product.dangdang.com/20426808.html "《网络管理必备工具软件精解（Linux版）》"