---
layout: post
title: FFmpeg 入门(2)：输出视频到屏幕
description: FFmpeg 入门教程系列。
category: blog
tag: FFmpeg, tutorial
---

## SDL

我们这里使用 SDL 来渲染视频到屏幕。SDL 是 Simple Direct Layer 的缩写，是一个优秀的跨平台多媒体库，你可以从 [http://www.libsdl.org][3] 下载 SDL 的库。


SDL 有很多可以将图像绘制都屏幕的方法，其中有一个专门用于将视频渲染到屏幕进行播放，即 YUV overlay。


YUV（其实这里叫 [YCbCr][4] 更准确）是不同于 RGB 的另一种存储原始图像数据的方式。简单来讲，Y 是表示「亮度」，U 和 V 表示「色度」，YUV 的关键是在于它的亮度信号 Y 和色度信号 U、V 是分离的。那就是说即使只有 Y 信号分量而没有 U、V 分量，我们仍然可以表示出图像，只不过图像是黑白灰度图像。在YCbCr 中 Y 是指亮度分量，Cb 指蓝色色度分量，而 Cr 指红色色度分量。

SDL 的 YUV overlay 可以接收一组 YUV 数据然后显示它。它支持 4 种不同的 YUV 格式，其中 「YV12」 是最快的。另一种 YUV 格式是 「YUV420P」也叫 「I420」，基本上和 「YV12」 是一样的，就是把 U 和 V 数组换了一下位置。

- YV12：亮度(行×列) ＋ U(行×列/4) + V(行×列/4)
- I420：亮度(行×列) ＋ V(行×列/4) + U(行×列/4)

更多 YUV420P 相关的信息，可以看这里：[Chroma subsampling][5]。

这里我们要做的是将上一个教程中的 `SaveFrame()` 函数去掉，换成渲染视频的逻辑。这里我们需要了解下给如何使用 SDL 库，首先我们需要 include SDL 库并初始化它：

```
#include <SDL.h>
#include <SDL_thread.h>

if (SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO | SDL_INIT_TIMER)) {
	fprintf(stderr, "Could not initialize SDL - %s\n", SDL_GetError());
	exit(1);
}
```

`SDL_Init()` 函数用来配置我们要用的功能特性，`SDL_GetError()` 是一个调试用的函数，用于输出错误信息。


## 创建显示界面

现在我们需要屏幕上的一块界面来渲染媒体视频，在 SDL 里这个界面叫做 `SDL_Surface`：

```
SDL_Surface *screen = NULL;

// Make a screen to put our video.
#ifndef __DARWIN__
screen = SDL_SetVideoMode(pCodecCtx->width, pCodecCtx->height, 0, 0);
#else
screen = SDL_SetVideoMode(pCodecCtx->width, pCodecCtx->height, 24, 0);
#endif
if (!screen) {
	fprintf(stderr, "SDL: could not set video mode - exiting\n");
	exit(1);
}
```

上面的代码中 `SDL_SetVideoMode()` 函数使用我们给定的宽高创建了一块界面，其中第三个参数是表示每个像素的比特数，如果为 0 则表示使用当前的显示一样的设置。但是在 OS X 上 0 是无效的，这里在 OS X 上设为 24。

接着，我们创建 YUV overlay 来输出我们的视频，同时这里我们使用 `SwsContext` 来将图像转换为 YUV420 格式：


```
struct SwsContext *sws_ctx = NULL;	
SDL_Overlay *bmp = NULL;

// Allocate a place to put our YUV image on that screen.
bmp = SDL_CreateYUVOverlay(pCodecCtx->width, pCodecCtx->height, SDL_YV12_OVERLAY, screen);

sws_ctx = sws_getContext(pCodecCtx->width, pCodecCtx->height, pCodecCtx->pix_fmt, pCodecCtx->width, pCodecCtx->height, AV_PIX_FMT_YUV420P, SWS_BILINEAR, NULL, NULL, NULL);
```


如我们前面所说，用 FFmpeg 从视频中获得 YUV420P 的数据，用 SDL 支持的 YV12 来进行渲染。



## 显示图像

现在我们要开始显示图像了，直接来看当我们获得完整的一帧图像时的代码：这里我们要创建一个 `AVFrame` 并将它的 `data` 和 `linesize` 的一组指针指向 YUV overlay：

```
// Did we get a video frame?
if (frameFinished) {
	SDL_LockYUVOverlay(bmp);

	AVFrame pict;
	pict.data[0] = bmp->pixels[0];
	pict.data[1] = bmp->pixels[2];
	pict.data[2] = bmp->pixels[1];

	pict.linesize[0] = bmp->pitches[0];
	pict.linesize[1] = bmp->pitches[2];
	pict.linesize[2] = bmp->pitches[1];

	// Convert the image into YUV format that SDL uses.
	sws_scale(sws_ctx, (uint8_t const * const *)pFrame->data, pFrame->linesize, 0, pCodecCtx->height, pict.data, pict.linesize);

	SDL_UnlockYUVOverlay(bmp);
}
```

由于我们需要向 YUV overlay 写入数据，这时候我们先给它加锁。`AVFrame` 这个结构体的 `data` 数组中存储的是 4 个通道指针，可以指向不同的通道，由于我们这里处理的 YUV420P 只有 Y、Cb、Cr 这 3 个通道，所以这里只有 3 组数据。处理其他格式时可能会遇到有 4 个通道数据的，比如 alpha 通道之类的。`linesize` 是每个通道的数据尺寸。在 YUV overlay 中与 data 和 linesize 对应的是 `pixels` 和 `pitches`（在 SDL 里 pitches 指的是一个 line 的数据的宽度）。我们这里做的便是将 `pict.data` 的 3 个数组指针指向 YUV overlay 对应的数据。这样一来，当我们向 `pict` 中写入数据时，其实我们是写到了 YUV overlay 中，overlay 中的内存则已经是分配好了的。我们在前面通过 `sws_ctx` 设置了目标格式为 `AV_PIX_FMT_YUV420P`，接着我们用 `sws_scale()` 函数来完成转换。 


## 渲染图像

接着我们要做的就是调用 `SDL_DisplayYUVOverlay()` 函数让 SDL 来渲染我们给它的数据，这时我们需要传入一个矩形区域数据告诉它从哪个点开始渲染，以及对应的宽度和高度。SDL 会帮我们做好缩放，并能够使用 GPU 来使图像渲染更快。


```
SDL_Rect rect;
if (frameFinished) {
	/* ... code ... */

	// Convert the image into YUV format that SDL uses.
	sws_scale(sws_ctx, (uint8_t const * const *)pFrame->data, pFrame->linesize, 0, pCodecCtx->height, pict.data, pict.linesize);

	SDL_UnlockYUVOverlay(bmp);

	rect.x = 0;
	rect.y = 0;
	rect.w = pCodecCtx->width;
	rect.h = pCodecCtx->height;
	SDL_DisplayYUVOverlay(bmp, &rect);
}

// Free the packet that was allocated by av_read_frame.
av_packet_unref(&packet);
```

到这里，我们的视频就可以显示了。


接下来，我们来看看 SDL 的另一个功能：事件系统。SDL 的事件系统使得你可以接收用户的输入，从而完成一些控制操作，在多线程编程时这个尤其有用。在这里，我们处理一个简单的 `SDL_QUIT` 事件，让我们可以退出程序。


```
SDL_Event event;

SDL_PollEvent(&event);
switch (event.type) {
	case SDL_QUIT:
		printf("SDL_QUIT\n");
		SDL_Quit();
		exit(0);
		break;
	default:
		break;
}
```

以上便是我们这节教程的全部代码，你可以从这里获得：[https://github.com/samirchen/TestFFmpeg][6]

## 编译执行

你可以使用下面的命令编译它：

```
$ gcc -o tutorial02 tutorial02.c -lavutil -lavformat -lavcodec -lswscale -lz -lm `sdl-config --cflags --libs`
```

找一个视频文件，你可以这样执行一下试试：

```
$ tutorial02 myvideofile.mp4
```


[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ffmpeg-tutorial-2
[3]: http://www.libsdl.org/
[4]: https://en.wikipedia.org/wiki/YCbCr
[5]: https://en.wikipedia.org/wiki/Chroma_subsampling
[6]: https://github.com/samirchen/TestFFmpeg

