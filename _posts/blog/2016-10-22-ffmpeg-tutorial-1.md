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

视频文件通常有一些基本结构。首先，文件本身被称为「容器(Container)」，容器的类型定义了文件的信息是如何存储。比如，AVI、QuickTime 等容器格式。





[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ffmpeg-tutorial-1
[3]: http://dranger.com/ffmpeg/tutorial01.html
[4]: https://github.com/samirchen/FFmpegCompileTool
[5]: http://www.samirchen.com/complie-ffmpeg-on-mac-os


