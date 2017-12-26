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


## 选择合适的缓冲算法


## 使用 HTTP DNS 加快建连









[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/video-playback-fast-play