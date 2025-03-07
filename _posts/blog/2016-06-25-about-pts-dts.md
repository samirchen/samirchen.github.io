---
layout: post
title: 理解音视频 PTS 和 DTS
description: 介绍音视频中 PTS、DTS 的概念。
category: blog
tag: Audio, Video, Live, FFmpeg, PTS, DTS
---


<!-- 
TS（Transport Stream）
PS（Packetized Stream）
ES（Elementary Stream）

PCR（Program Clock Reference）
STC（System Time Clock）
-->

## 视频

视频的播放过程可以简单理解为一帧一帧的画面按照时间顺序呈现出来的过程，就像在一个本子的每一页画上画，然后快速翻动的感觉。

![image](../../images/about-pts-dts/animation-notebook.gif)

但是在实际应用中，并不是每一帧都是完整的画面，因为如果每一帧画面都是完整的图片，那么一个视频的体积就会很大，这样对于网络传输或者视频数据存储来说成本太高，所以通常会对视频流中的一部分画面进行压缩（编码）处理。由于压缩处理的方式不同，视频中的画面帧就分为了不同的类别，其中包括：I 帧、P 帧、B 帧。



## I、P、B 帧

I 帧、P 帧、B 帧的区别在于：

- I 帧（Intra coded frames）：I 帧图像采用帧内编码方式，即只利用了单帧图像内的空间相关性，而没有利用时间相关性。I 帧使用帧内压缩，不使用运动补偿，由于 I 帧不依赖其它帧，所以是随机存取的入点，同时是解码的基准帧。I 帧主要用于接收机的初始化和信道的获取，以及节目的切换和插入，I 帧图像的压缩倍数相对较低。I 帧图像是周期性出现在图像序列中的，出现频率可由编码器选择。
- P 帧（Predicted frames）：P 帧和 B 帧图像采用帧间编码方式，即同时利用了空间和时间上的相关性。P 帧图像只采用前向时间预测，可以提高压缩效率和图像质量。P 帧图像中可以包含帧内编码的部分，即 P 帧中的每一个宏块可以是前向预测，也可以是帧内编码。
- B 帧（Bi-directional predicted frames）：B 帧图像采用双向时间预测，可以大大提高压缩倍数。值得注意的是，由于 B 帧图像采用了未来帧作为参考，因此 MPEG-2 编码码流中图像帧的传输顺序和显示顺序是不同的。

也就是说，一个 I 帧可以不依赖其他帧就解码出一幅完整的图像，而 P 帧、B 帧不行。P 帧需要依赖视频流中排在它前面的帧才能解码出图像。B 帧则需要依赖视频流中排在它前面或后面的帧才能解码出图像。

这就带来一个问题：在视频流中，先到来的 B 帧无法立即解码，需要等待它依赖的后面的 I、P 帧先解码完成，这样一来播放时间与解码时间不一致了，顺序打乱了，那这些帧该如何播放呢？这时就需要我们来了解另外两个概念：DTS 和 PTS。




## DTS、PTS 的概念

DTS、PTS 的概念如下所述：

- DTS（Decoding Time Stamp）：即解码时间戳，这个时间戳的意义在于告诉播放器该在什么时候解码这一帧的数据。
- PTS（Presentation Time Stamp）：即显示时间戳，这个时间戳用来告诉播放器该在什么时候显示这一帧的数据。

需要注意的是：虽然 DTS、PTS 是用于指导播放端的行为，但它们是在编码的时候由编码器生成的。

当视频流中没有 B 帧时，通常 DTS 和 PTS 的顺序是一致的。但如果有 B 帧时，就回到了我们前面说的问题：解码顺序和播放顺序不一致了。

比如一个视频中，帧的显示顺序是：I B B P，现在我们需要在解码 B 帧时知道 P 帧中信息，因此这几帧在视频流中的顺序可能是：I P B B，这时候就体现出每帧都有 DTS 和 PTS 的作用了。DTS 告诉我们该按什么顺序解码这几帧图像，PTS 告诉我们该按什么顺序显示这几帧图像。顺序大概如下：

```
   PTS: 1 4 2 3
   DTS: 1 2 3 4
Stream: I P B B
```




## 音视频的同步 

上面说了视频帧、DTS、PTS 相关的概念。我们都知道在一个媒体流中，除了视频以外，通常还包括音频。音频的播放，也有 DTS、PTS 的概念，但是音频没有类似视频中 B 帧，不需要双向预测，所以音频帧的 DTS、PTS 顺序是一致的。

音频视频混合在一起播放，就呈现了我们常常看到的广义的视频。在音视频一起播放的时候，我们通常需要面临一个问题：怎么去同步它们，以免出现画不对声的情况。

要实现音视频同步，通常需要选择一个参考时钟，参考时钟上的时间是线性递增的，编码音视频流时依据参考时钟上的时间给每帧数据打上时间戳。在播放时，读取数据帧上的时间戳，同时参考当前参考时钟上的时间来安排播放。这里的说的时间戳就是我们前面说的 PTS。实践中，我们可以选择：同步视频到音频、同步音频到视频、同步音频和视频到外部时钟。



更多的音视频技术文章可以去看：[关键帧Keyframe：https://gjzkeyframe.github.io/](https://gjzkeyframe.github.io/)

## 参考

- [MPEG-2 Wiki][3]
- [MPEG-2 的同步及时间恢复][4]
- [Synching Video][5]

[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/about-pts-dts
[3]: https://zh.wikipedia.org/wiki/MPEG-2
[4]: http://blog.csdn.net/hice1226/article/details/6717354
[5]: http://dranger.com/ffmpeg/tutorial05.html

