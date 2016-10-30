---
layout: post
title: FFmpeg 入门(1)：截取视频帧
description: FFmpeg 入门教程系列。
category: blog
tag: FFmpeg, tutorial
---

## 背景

在 Mac OS 上如果要运行教程中的相关代码需要先安装 FFmpeg，建议使用 brew 来安装：

```
// 用 brew 安装 FFmpeg：
brew install ffmpeg
```

或者你可以参考[在 Mac OS 上编译 FFmpeg][5]使用源码编译和安装 FFmpeg。

教程原文地址：[http://dranger.com/ffmpeg/tutorial01.html][3]，本文中的代码做过部分修正。 



## 概要

视频文件通常有一些基本的组成部分。首先，文件本身被称为**「容器(container)」**，容器的类型定义了文件的信息是如何存储，比如，AVI、QuickTime 等容器格式。接着，你需要了解的概念是**「流(streams)」**，例如，你通常会有一路音频流和一路视频流。流中的数据元素被称为**「帧(frames)」**。每路流都会被相应的**「编/解码器(codec)」**进行编码或解码（codec 这个名字就是源于 COded 和 DECoded）。codec 定义了实际数据是如何被编解码的，比如你用到的 codecs 可能是 DivX 和 MP3。「数据包(packets)」是从流中读取的数据片段，这些数据片段中包含的一个个比特就是解码后能最终被我们的应用程序处理的原始帧数据。为了达到我们音视频处理的目标，每个数据包都包含着完整的帧，在音频情况下，一个数据包中可能会包含多个音频帧。

基于以上这些基础，处理视频流和音频流的过程其实很简单：

- 1：从 video.avi 文件中打开 video_stream。
- 2：从 video_stream 中读取数据包到 frame。
- 3：如果数据包中的 frame 不完整，则跳到步骤 2。
- 4：处理 frame。
- 5：跳到步骤 2。

尽管在一些程序中上面步骤 4 处理 frame 的逻辑可能会非常复杂，但是在本文中的例程中，用 FFmpeg 来处理多媒体文件的部分会写的比较简单一些，这里我们将要做的就是打开一个视频文件，读取其中的视频流，将视频流中获取到的视频帧写入到 PPM 文件中保存起来。

下面我们一步一步来实现。

## 打开文件




[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ffmpeg-tutorial-1
[3]: http://dranger.com/ffmpeg/tutorial01.html
[4]: https://github.com/samirchen/FFmpegCompileTool
[5]: http://www.samirchen.com/complie-ffmpeg-on-mac-os


