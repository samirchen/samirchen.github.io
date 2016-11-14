---
layout: post
title: FFmpeg 入门(4)：线程分治
description: FFmpeg 入门教程系列。
category: blog
tag: FFmpeg, tutorial
---


## 概览

上一节教程中，我们使用 SDL 的音频相关的函数来支持音频播放。SDL 起了一个线程来在需要音频数据的时候去调用我们定义的回调方法。现在我们要做的是用线程的方法去改造视频显示这块的逻辑。这样一来会使得代码的机构更模块化，这样改动起来会更简单，尤其是当我们想添加音视频同步逻辑时。

我们从哪开始呢？首先，我们发现我们的 main 函数做的事情太多了：运行 event loop、读取数据包、进行视频解码等等，所以我们现在要做的就是把这些事情拆分掉：一个线程来专门做数据包解码，这些数据包会被添加到队列中，并分别被音频处理线程和视频处理线程来读取和处理。音频处理线程我们在上一节已经按照设想写好了，视频处理线程要复杂一些，因为我们需要自己来显示视频。我们要把显示视频的代码放到 main loop，但是不会每次循环的时候去做显示，而是把视频显示集成 event loop 中去。方式就是解码视频数据，把视频帧存到另一个队列，然后创建一个自定义事件(FF_REFRESH_EVENT)添加到事件系统，然后当 event loop 处理这个事件时就会显示队列中的下一帧。下面我们用个图来说明下这个流程：



```
 ________ audio  _______      _____
|        | pkts |       |    |     | to speaker.
| DECODE |----->| AUDIO |--->| SDL |-->
|________|      |_______|    |_____|
    |  video     _______
    |   pkts    |       |
    +---------->| VIDEO |
 ________       |_______|   _______
|       |          |       |       |
| EVENT |          +------>| VIDEO | to monitor.
| LOOP  |----------------->| DISP. |-->
|_______|<---FF_REFRESH----|_______|
```


把视频显示的逻辑放到 event loop 中去的主要目的是为了使用 `SDL_Delay` 线程，这样我们可以准确的控制下一个视频帧什么时候显示到屏幕上。在下一节教程中当我们要做音视频同步时，相关的逻辑就会变得简单些了。


## 精简代码

我们现在要创建一个比较大的结构体来容纳所有音视频信息，叫做 `VideoState`。

```
typedef struct VideoState {
	AVFormatContext *pFormatCtx;
	int videoStream, audioStream;
	AVStream *audio_st;
	PacketQueue audioq;
	uint8_t audio_buf[(MAX_AUDIO_FRAME_SIZE * 3) / 2];
	unsigned int audio_buf_size;
	unsigned int audio_buf_index;
	AVFrame audio_frame;
	AVPacket audio_pkt;
	uint8_t *audio_pkt_data;
	int audio_pkt_size;
	AVStream *video_st;
	PacketQueue videoq;
	
	VideoPicture pictq[VIDEO_PICTURE_QUEUE_SIZE];
	int pictq_size, pictq_rindex, pictq_windex;
	SDL_mutex *pictq_mutex;
	SDL_cond *pictq_cond;
	
	SDL_Thread *parse_tid;
	SDL_Thread *video_tid;
	
	char filename[1024];
	int quit;
	
	AVIOContext *io_context;
	struct SwsContext *sws_ctx;
} VideoState;
```







以上便是我们这节教程的全部代码，你可以从这里获得：[https://github.com/samirchen/TestFFmpeg][6]

## 编译执行

你可以使用下面的命令编译它：

```
$ gcc -o tutorial04 tutorial04.c -lavutil -lavformat -lavcodec -lswscale -lz -lm `sdl-config --cflags --libs`
```

找一个视频文件，你可以这样执行一下试试：

```
$ tutorial04 myvideofile.mp4
```



[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ffmpeg-tutorial-4