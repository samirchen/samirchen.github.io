layout: post
title: 点播中的播放成功率优化
description: 介绍在点播中进行播放成功率优化的一些思路。
category: blog
tag: Audio, Video, MP4, Error
---



## 网络重连

网络抖动造成建连失败是影响播放成功率的一种重要原因。通常因为网络原因而引起播放错误时，播放器会上报相应的错误码，这时候我们可以根据这些错误码针对性地对播放器进行刷新，重连来进行网络连接，并设置重试次数限制，通过这样的方式来尝试恢复播放。

网络重连的实现可以是在播放器底层的网络连接相关的模块，也可以是在播放器上层重新刷新整个播放器生命周期来完成。



## 解码方式切换

对于基于 FFmpeg 实现的播放器，在播放视频时，我们可以选择硬解或者软解的方式来对音视频数据进行解码。有时候如果遇到硬件对视频格式支持的不够完善，可能会出现解码出错，这时候我们可以通过上报上来的错误探测是否是解码出错，然后针对性地选择切换解码方式来尝试恢复播放。要么硬解切换到软解，要么软解切换到硬解。

一般而言，软解的稳定性要好于硬解，毕竟对硬解的支持会因为设备而变。但是，软解对于 CPU、内存的消耗会更大。



## 视频格式规范

有时候播放错误是因为播放器对视频格式支持的不够完善而造成的。由于生产视频的设备各异，也就造成了视频的格式有着不同的标准。

对于点播视频而言，在允许的情况下，我们应该在服务端来尽量规范视频格式，比如我们可以配合播放器的实现对上传到服务端的视频进行统一格式的重编码，这样可以尽量减少播放器端因为对视频格式支持不够完善而引起播放错误。



## 工具

下面是一些可以用来分析视频信息的工具：

- ffplay
- ffprobe
- mediainfo
- hls-analyzer
- [http://www.cutv.com/demo/live_test.swf][3]
- [http://www.ossrs.net/players/srs_player.html][4]
- VLC
- vplayer
- mxplayer
- mp4info
- FlvParse
- FLVMeta
- Elecard StreamEye Studio






[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/video-playback-error
[3]: http://www.cutv.com/demo/live_test.swf
[4]: http://www.ossrs.net/players/srs_player.html

