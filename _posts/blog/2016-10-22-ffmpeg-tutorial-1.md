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

媒体文件通常有一些基本的组成部分。首先，文件本身被称为**「容器(container)」**，容器的类型定义了文件的信息是如何存储，比如，AVI、QuickTime 等容器格式。接着，你需要了解的概念是**「流(streams)」**，例如，你通常会有一路音频流和一路视频流。流中的数据元素被称为**「帧(frames)」**。每路流都会被相应的**「编/解码器(codec)」**进行编码或解码（codec 这个名字就是源于 COded 和 DECoded）。codec 定义了实际数据是如何被编解码的，比如你用到的 codecs 可能是 DivX 和 MP3。**「数据包(packets)」**是从流中读取的数据片段，这些数据片段中包含的一个个比特就是解码后能最终被我们的应用程序处理的原始帧数据。为了达到我们音视频处理的目标，每个数据包都包含着完整的帧，在音频情况下，一个数据包中可能会包含多个音频帧。

基于以上这些基础，处理视频流和音频流的过程其实很简单：

- 1：从 video.avi 文件中打开 video_stream。
- 2：从 video_stream 中读取数据包到 frame。
- 3：如果数据包中的 frame 不完整，则跳到步骤 2。
- 4：处理 frame。
- 5：跳到步骤 2。

尽管在一些程序中上面步骤 4 处理 frame 的逻辑可能会非常复杂，但是在本文中的例程中，用 FFmpeg 来处理多媒体文件的部分会写的比较简单一些，这里我们将要做的就是打开一个媒体文件，读取其中的视频流，将视频流中获取到的视频帧写入到 PPM 文件中保存起来。

下面我们一步一步来实现。

## 打开媒体文件

首先，我们来看看如何打开媒体文件。在使用 FFmpeg 时，首先需要初始化对应的 Library。


```
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libswscale/swscale.h>
#include <libavutil/imgutils.h>
//...

int main(int argc, char *argv[]) {

	// Register all formats and codecs.
	av_register_all();

	// ...
}

```

上面的代码会注册 FFmpeg 库中所有可用的「视频格式」和 「codec」，这样当使用库打开一个媒体文件时，就能找到对应的视频格式处理程序和 codec 来处理。需要注意的是在使用 FFmpeg 时，你只需要调用 `av_register_all()` 一次即可，因此我们在 main 中调用。当然，你也可以根据需求只注册给定的视频格式和 codec，但通常你不需要这么做。


接下来我们就要准备打开媒体文件了，那么媒体文件中有哪些信息是值得注意的呢？

- 是否包含：音频、视频。
- 码流的封装格式，用于解封装。
- 视频的编码格式，用于初始化视频解码器
- 音频的编码格式，用于初始化音频解码器。
- 视频的分辨率、帧率、码率，用于视频的渲染。
- 音频的采样率、位宽、通道数，用于初始化音频播放器。
- 码流的总时长，用于展示、拖动 Seek。
- 其他 Metadata 信息，如作者、日期等，用于展示。

这些关键的媒体信息，被称作 **metadata**，常常记录在整个码流的开头或者结尾处，例如：wav 格式主要由 wav header 头来记录音频的采样率、通道数、位宽等关键信息；mp4 格式，则存放在 moov box 结构中；而 FLV 格式则记录在 onMetaData 中等等。


`avformat_open_input` 这个函数主要负责服务器的连接和码流头部信息的拉取，我们就用它来打开媒体文件：

```
AVFormatContext *pFormatCtx = NULL;

// Open video file.
if (avformat_open_input(&pFormatCtx, argv[1], NULL, NULL) != 0) {
	return -1; // Couldn't open file.
}
```

我们从程序入口获得要打开文件的路径，作为 `avformat_open_input` 函数的第二个参数传入，这个函数会读取媒体文件的文件头并将文件格式相关的信息存储在我们作为第一个参数传入的 `AVFormatContext` 数据结构中。`avformat_open_input` 函数的第三个参数用于指定媒体文件格式，第四个参数是文件格式相关选项。如果你后面这两个参数传入的是 NULL，那么 libavformat 将自动探测文件格式。

接下来对于媒体信息的探测和分析工作就要交给 `avformat_find_stream_info` 函数了：

```
// Retrieve stream information.
if (avformat_find_stream_info(pFormatCtx, NULL) < 0) {
	return -1; // Couldn't find stream information.
}
```

`avformat_find_stream_info` 函数会为 `pFormatCtx->streams` 填充对应的信息。这里还有一个调试用的函数 `av_dump_format` 可以为我们打印 `pFormatCtx` 中都有哪些信息。

```
// Dump information about file onto standard error.
av_dump_format(pFormatCtx, 0, argv[1], 0);
```

`AVFormatContext` 里包含了下面这些跟媒体信息有关的成员：

- struct AVInputFormat *iformat; // 记录了封装格式信息
- unsigned int nb_streams; // 记录了该 URL 中包含有几路流
- AVStream **streams; // 一个结构体数组，每个对象记录了一路流的详细信息
- int64_t start_time; // 第一帧的时间戳
- int64_t duration; // 码流的总时长
- int bit_rate; // 码流的总码率，bps
- AVDictionary *metadata; // 一些文件信息头，key/value 字符串

由此可见，经过 `avformat_find_stream_info` 的处理，我们可以拿到媒体资源的封装格式、总时长、总码率了。此外 `pFormatCtx->streams` 是一个 `AVStream` 指针的数组，里面包含了媒体资源的每一路流信息，数组的大小为 `pFormatCtx->nb_streams`。

`AVStream` 结构体中关键的成员包括：

- AVCodecContext *codec; // 记录了该码流的编码信息
- int64_t start_time; // 第一帧的时间戳
- int64_t duration; // 该码流的时长
- int64_t nb_frames; // 该码流的总帧数
- AVDictionary *metadata; // 一些文件信息头，key/value 字符串
- AVRational avg_frame_rate; // 平均帧率

这里可以拿到平均帧率。

`AVCodecContext` 则记录了一路流的具体编码信息，其中关键的成员包括：

- const struct AVCodec *codec; // 编码的详细信息
- enum AVCodecID codec_id; // 编码类型
- int bit_rate; // 平均码率
- video only：
	- int width, height; // 图像的宽高尺寸，码流中不一定存在该信息，会由解码后覆盖
	- enum AVPixelFormat pix_fmt; // 原始图像的格式，码流中不一定存在该信息，会由解码后覆盖
- audio only：
	- int sample_rate; // 音频的采样率
	- int channels; // 音频的通道数
	- enum AVSampleFormat sample_fmt; // 音频的格式，位宽
	- int frame_size; // 每个音频帧的 sample 个数

可以看到编码类型、图像的宽度高度、音频的参数都在这里了。


了解完这些数据结构，我们接着往下走，直到我们找到一个视频流：


```
// Find the first video stream.
videoStream = -1;
for (i = 0; i < pFormatCtx->nb_streams; i++) {
	if(pFormatCtx->streams[i]->codec->codec_type == AVMEDIA_TYPE_VIDEO) {
		videoStream = i;
		break;
	}
}
if (videoStream == -1) {
	return -1; // Didn't find a video stream.
}

// Get a pointer to the codec context for the video stream.
pCodecCtxOrig = pFormatCtx->streams[videoStream]->codec;
```

流信息中关于 codec 的部分存储在 codec context 中，这里包含了这路流所使用的所有的 codec 的信息，现在我们有一个指向它的指针了，但是我们接着还需要找到真正的 codec 并打开它：

```
// Find the decoder for the video stream.
pCodec = avcodec_find_decoder(pCodecCtxOrig->codec_id);
if (pCodec == NULL) {
	fprintf(stderr, "Unsupported codec!\n");
	return -1; // Codec not found.
}
// Copy context.
pCodecCtx = avcodec_alloc_context3(pCodec);
if (avcodec_copy_context(pCodecCtx, pCodecCtxOrig) != 0) {
	fprintf(stderr, "Couldn't copy codec context");
	return -1; // Error copying codec context.
}

// Open codec.
if (avcodec_open2(pCodecCtx, pCodec, NULL) < 0) {
	return -1; // Could not open codec.
}
```


需要注意，我们不能直接使用视频流中的 `AVCodecContext`，所以我们需要用 `avcodec_copy_context()` 来拷贝一份新的 `AVCodecContext` 出来。



## 存储数据

接下来，我们需要一个地方来存储视频中的帧：

```
AVFrame *pFrame = NULL;

// Allocate video frame.
pFrame = av_frame_alloc();
```


由于我们计划将视频帧输出存储为 PPM 文件，而 PPM 文件是会存储为 24-bit RGB 格式的，所以我们需要将视频帧从它本来的格式转换为 RGB。FFmpeg 可以帮我们做这些。对于大多数的项目，我们可能都有将原来的视频帧转换为指定格式的需求。现在我们就来创建一个`AVFrame` 用于格式转换：

```
// Allocate an AVFrame structure.
pFrameRGB = av_frame_alloc();
if (pFrameRGB == NULL) {
	return -1;
}
```


尽管我们已经分配了内存类处理视频帧，当我们转格式时，我们仍然需要一块地方来存储视频帧的原始数据。我们使用 `av_image_get_buffer_size` 来获取需要的内存大小，然后手动分配这块内存。


```
int numBytes;
uint8_t *buffer = NULL;

// Determine required buffer size and allocate buffer.
numBytes = av_image_get_buffer_size(AV_PIX_FMT_RGB24, pCodecCtx->width, pCodecCtx->height, 1);
buffer = (uint8_t *) av_malloc(numBytes * sizeof(uint8_t));
```

`av_malloc` 是一个 FFmpeg 的 malloc，主要是对 `malloc` 做了一些封装来保证地址对齐之类的事情，它不会保证你的代码不发生内存泄漏、多次释放或其他 malloc 问题。


现在我们用 `av_image_fill_arrays` 函数来关联 frame 和我们刚才分配的内存。

```
// Assign appropriate parts of buffer to image planes in pFrameRGB Note that pFrameRGB is an AVFrame, but AVFrame is a superset of AVPicture
av_image_fill_arrays(pFrameRGB->data, pFrameRGB->linesize, buffer, AV_PIX_FMT_RGB24, pCodecCtx->width, pCodecCtx->height, 1);
```

现在，我们准备从视频流读取数据了。


## 读取数据


接下来我们要做的就是从整个视频流中读取数据包 packet，并将数据解码到我们的 frame 中，一旦获得完整的 frame，我们就转换其格式并存储它。

``` 
AVPacket packet;
int frameFinished;
struct SwsContext *sws_ctx = NULL;

// Initialize SWS context for software scaling.
sws_ctx = sws_getContext(pCodecCtx->width, pCodecCtx->height, pCodecCtx->pix_fmt, pCodecCtx->width, pCodecCtx->height, AV_PIX_FMT_RGB24, SWS_BILINEAR, NULL, NULL, NULL);

// Read frames and save first five frames to disk.
i = 0;
while (av_read_frame(pFormatCtx, &packet) >= 0) {
	// Is this a packet from the video stream?
	if (packet.stream_index == videoStream) {
		// Decode video frame
		avcodec_decode_video2(pCodecCtx, pFrame, &frameFinished, &packet);

		// Did we get a video frame?
		if (frameFinished) {
			// Convert the image from its native format to RGB.
			sws_scale(sws_ctx, (uint8_t const * const *) pFrame->data, pFrame->linesize, 0, pCodecCtx->height, pFrameRGB->data, pFrameRGB->linesize);

			// Save the frame to disk.
			if (++i <= 5) {
				SaveFrame(pFrameRGB, pCodecCtx->width, pCodecCtx->height, i);
			}
		}
	}

	// Free the packet that was allocated by av_read_frame.
	av_packet_unref(&packet);
}
```

接下来的程序是比较好理解的：`av_read_frame()` 函数从视频流中读取一个数据包 packet，把它存储在 `AVPacket` 数据结构中。需要注意，我们只创建了 packet 结构，FFmpeg 则为我们填充了其中的数据，其中 `packet.data` 这个指针会指向这些数据，而这些数据占用的内存需要通过 `av_packet_unref()` 函数来释放。`avcodec_decode_video2()` 函数将数据包 packet 转换为视频帧 frame。但是，我们可能无法通过只解码一个 packet 就获得一个完整的视频帧 frame，可能需要读取多个 packet 才行，`avcodec_decode_video2()` 会在解码到完整的一帧时设置 `frameFinished` 为真。最后当解码到完整的一帧时，我们用 `sws_scale()` 函数来将视频帧本来的格式 `pCodecCtx->pix_fmt` 转换为 RGB。记住你可以将一个 `AVFrame` 指针转换为一个 `AVPicture` 指针。最后，我们使用我们的 `SaveFrame` 函数来保存这一个视频帧到文件。

在 `SaveFrame` 函数中，我们将 RGB 信息写入到一个 PPM 文件中。

```
void SaveFrame(AVFrame *pFrame, int width, int height, int iFrame) {
	FILE *pFile;
	char szFilename[32];
	int y;
  
	// Open file.
	sprintf(szFilename, "frame%d.ppm", iFrame);
	pFile = fopen(szFilename, "wb");
	if (pFile == NULL) {
		return;
	}
  
	// Write header.
	fprintf(pFile, "P6\n%d %d\n255\n", width, height);
  
	// Write pixel data.
	for (y = 0; y < height; y++) {
		fwrite(pFrame->data[0]+y*pFrame->linesize[0], 1, width*3, pFile);
	}
  
	// Close file.
	fclose(pFile);
}
```

下面我们回到 main 函数，当我们完成了视频流的读取，我们需要做一些扫尾工作：

```
// Free the RGB image.
av_free(buffer);
av_frame_free(&pFrameRGB);

// Free the YUV frame.
av_frame_free(&pFrame);

// Close the codecs.
avcodec_close(pCodecCtx);
avcodec_close(pCodecCtxOrig);

// Close the video file.
avformat_close_input(&pFormatCtx);

return 0;
```

你可以看到，这里我们用 `av_free()` 函数来释放我们用 `av_malloc()` 分配的内存。


以上便是我们这节教程的全部代码，你可以从这里获得：[https://github.com/samirchen/TestFFmpeg][6]


## 编译执行

你可以使用下面的命令编译它：

```
$ gcc -o tutorial01 tutorial01.c -lavutil -lavformat -lavcodec -lswscale -lz -lm
```

找一个媒体文件，你可以这样执行一下试试：

```
$ tutorial01 myvideofile.mp4
```


[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ffmpeg-tutorial-1
[3]: http://dranger.com/ffmpeg/tutorial01.html
[4]: https://github.com/samirchen/FFmpegCompileTool
[5]: http://www.samirchen.com/complie-ffmpeg-on-mac-os
[6]: https://github.com/samirchen/TestFFmpeg
[7]: http://ticktick.blog.51cto.com/823160/1869849

