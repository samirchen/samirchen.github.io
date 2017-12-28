---
layout: post
title: 点播中的首屏秒开优化
description: 介绍在点播中进行首屏秒开优化的一些思路。
category: blog
tag: Audio, Video, MP4, Fast Play
---


## 前置 metadata

播放器在网络点播场景下去请求 MP4 视频数据，需要先获取到文件的 metadata，解析出该文件的编码、帧率等信息后才能开始边下边播。如果 MP4 的 metadata 数据块被编码在文件尾部，这种情况会导致播放器只有下载完整个文件后才能成功解析并播放这个视频。对于这种视频，我们最好能够在服务端将其重新编码，将 metadata 数据块转移到靠近文件头部的位置，保证播放器在线请求时能较快播放。比如 FFmpeg 的下列命令就可以支持这个操作：

```
ffmpeg -i bad.mp4 -movflags faststart good.mp4
```



## 优化 FFmpeg 的播放参数

基于 FFmpeg 实现的播放器，在播放视频时都会调用到一个 `avformat_find_stream_info` (libavformat/utils.c) 函数，该函数的作用是读取一定长度的码流数据，来分析码流的基本信息，为视频中各个媒体流的 AVStream 结构体填充好相应的数据。这个函数中做了查找合适的解码器、打开解码器、读取一定的音视频帧数据、尝试解码音视频帧等工作，基本上完成了解码的整个流程。在不清楚视频数据的格式又要做到较好的兼容性时，这个过程是比较耗时的，从而会影响到播放器首屏秒开。

在外部可以通过设置 `probesize` 和 `analyzeduration` 两个参数来控制该函数读取的数据量大小和分析时长为比较小的值来降低 `avformat_find_stream_info` 的耗时，从而优化播放器首屏秒开。但是，需要注意的是这两个参数设置过小时，可能会造成预读数据不足，无法解析出码流信息，从而导致播放失败、无音频或无视频的情况。所以，在服务端对视频格式进行标准化转码，从而确定视频格式，进而再去推算 `avformat_find_stream_info` 分析码流信息所兼容的最小的 `probesize` 和 `analyzeduration`，就能在保证播放成功率的情况下最大限度地区优化首屏秒开。

在我们能控制视频格式达到标准化后，我们可以直接修改 `avformat_find_stream_info` 的实现逻辑，针对该视频格式做优化，进而优化首屏秒开。比如，你可以试试将函数中用到的一个变量 `fps_analyze_framecount` 初始化为 0 试试效果。

甚至，我们可以进一步直接去掉 `avformat_find_stream_info` 这个过程，自定义完成解码环境初始化。参见：[VLC优化（1） avformat_find_stream_info 接口延迟降低][4] 和 [FFMPEG avformat_find_stream_info 替换][5]。


对 `avformat_find_stream_info` 代码的分析，还可以看看这里：[FFmpeg源代码简单分析：avformat_find_stream_info()][3]。




## 选择合适的缓冲算法和缓冲区

在点播场景下，为了减少播放过程中的卡顿，通常会缓冲一定的数据后再解码播放，这是一种播放策略。

为了加快首屏播放速度，也可以选择降低首次缓冲的数据量。甚至在第一帧没有渲染出来的情况下，不做任何缓冲，有数据就直接塞给解码器解码播放。

在 iOS 平台上，使用系统的 AVPlayer 时，属性 `automaticallyWaitsToMinimizeStalling` 就是控制播放器缓冲策略的。当该值为 YES 时，AVPlayer 会努力尝试延迟开始播放，加载足够的数据来保证整个播放过程中尽量卡顿最少。这个接口在 iOS 10 及以上版本才开放，在 iOS 10 之前的版本，在播放 HLS 这种流媒体视频时，效果如同 `automaticallyWaitsToMinimizeStalling` 为 YES，播放基于文件的视频资源，包括通过网络传输的网络视频文件，则效果如同 `automaticallyWaitsToMinimizeStalling` 为 NO。




## 使用 HTTP DNS 加快建连









[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/video-playback-fast-play
[3]: http://blog.csdn.net/leixiaohua1020/article/details/44084321
[4]: https://jiya.io/archives/vlc_optimize_1.html
[5]: http://blog.csdn.net/leo2007608/article/details/53421528

