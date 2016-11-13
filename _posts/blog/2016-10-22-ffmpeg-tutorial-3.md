---
layout: post
title: FFmpeg 入门(3)：播放音频
description: FFmpeg 入门教程系列。
category: blog
tag: FFmpeg, tutorial
---

## 音频

SDL 提供了播放音频的方法。`SDL_OpenAudio` 函数用来让设备播放音频，它需要我们传入一个包含了所有我们输出需要的音频信息的 `SDL_AudioSpec` 结构体数据。

在展示接下来的代码之前，我们先说说 PC 上是如何处理音频的。数字音频包含了一长串**「音频采样(sample)」**，每一个采样代表着一个音频波形的值。声音是在一定的**「音频采样率(sample rate)」**下被录制下来的，音频采样率即每秒音频采样的数量，表示的是播放音频速度。常见的音频采样率是 22500 和 44100，分别用于广播和 CD。此外，大部分音频还可以用更多的通道来实现立体声和环绕声等效果，比如立体声会一次来 2 个音频采样。这样当我们从媒体文件中获取数据时，我们不知道我们会获得多少音频采样，而 FFmpeg 也不会只给我们部分采样，也就是说，它不会对立体声的多通道采样进行分割。


SDL 的音频播放的实现大致是这样的：创建 `SDL_AudioSpec` 结构体，设置你的音频播放数据，包括：采样率(freq)、音频格式(format)、通道数(channels)、采样大小(samples)、回调函数(callback)和用户数据(userdata)等。当开始播放音频时，SDL 会持续调用这个回调方法来填充固定数量的字节到音频缓冲区。然后我们调用 `SDL_OpenAudio()` 函数，传入这个 `SDL_AudioSpec` 结构体数据，这时它会打开音频设备并给我们返回一个另外的 `SDL_AudioSpec`，这个才是我们真正使用的 `SDL_AudioSpec`，这个跟我们传入的可能有不同。


## 创建音频

简单介绍了音频相关的知识后，我们接下来开始看代码：像处理视频流一样，我们从媒体文件中获取音频流。

```
// Find the first video stream.
videoStream = -1;
audioStream = -1;
for (i = 0; i < pFormatCtx->nb_streams; i++) {
	if (pFormatCtx->streams[i]->codec->codec_type == AVMEDIA_TYPE_VIDEO && videoStream < 0) {
		videoStream = i;
	}
	if (pFormatCtx->streams[i]->codec->codec_type == AVMEDIA_TYPE_AUDIO && audioStream < 0) {
		audioStream = i;
	}
}
if (videoStream == -1) {
	return -1; // Didn't find a video stream.
}
if (audioStream == -1) {
	return -1; // Didn't find a audio stream.
}
```

接着我们可以从 `AVStream` 的中 `AVCodecContext` 结构体中获取我们想要的所有信息。拿到 `AVCodecContext` 后，我们就可以用这些信息来创建音频了：


```
AVCodecContext *aCodecCtx = NULL;
AVCodec *aCodec = NULL;
SDL_AudioSpec wanted_spec, spec;

aCodecCtx = pFormatCtx->streams[audioStream]->codec;
aCodec = avcodec_find_decoder(aCodecCtx->codec_id);
if (!aCodec) {
	fprintf(stderr, "Unsupported codec!\n");
	return -1;
}

// Set audio settings from codec info.
wanted_spec.freq = aCodecCtx->sample_rate;
wanted_spec.format = AUDIO_S16SYS;
wanted_spec.channels = aCodecCtx->channels;
wanted_spec.silence = 0;
wanted_spec.samples = SDL_AUDIO_BUFFER_SIZE;
wanted_spec.callback = audio_callback;
wanted_spec.userdata = aCodecCtx;

if (SDL_OpenAudio(&wanted_spec, &spec) < 0) {
	fprintf(stderr, "SDL_OpenAudio: %s\n", SDL_GetError());
	return -1;
}

avcodec_open2(aCodecCtx, aCodec, &audioOptionsDict);
```


对于 `SDL_AudioSpec` 的成员，我们这里说明一下：

- freq: 采样率。
- format: 这个参数用来告诉 SDL 音频的格式。`S16SYS` 中的 `S` 表示 `signed`，`16` 表示每个采样占 16 bits。`SYS` 表示大小端和系统保持一致。这个格式是 `avcodec_decode_audio4()` 会给我们的。
- channels: 音频通道数。
- silence: 是否静音。
- samples: 音频缓存的大小。推荐值为 512~8192，ffplay 使用的是 1024。
- callback: 回调函数。
- userdata: 回调函数带的用户数据。

最后我们通过 `SDL_OpenAudio()` 来打开音频。


## 队列

现在我们可以从码流里拉取音频数据了，但是我们接下来改如何处理这些数据呢？我们持续从媒体文件中获取数据包(packet)，与此同时，SDL 也会持续调用回调方法。一种解决方案时，创建一块全局的存储区，让我们能不断把数据放进去，让 SDL 能不断通过 `audio_callback` 从里面把数据取出来作进一步处理。所以接下来我们将创建一个数据包的队列(packet queue)。事实上，FFmpeg 已经提供了相应的数据结构 `AVPacketList`，这是一个 packet 链表。基于此，我们定义了我们的 `PacketQueue`：

```
typedef struct PacketQueue {
	AVPacketList *first_pkt, *last_pkt;
	int nb_packets;
	int size;
	SDL_mutex *mutex;
	SDL_cond *cond;
} PacketQueue;
```

需要指出的是 `nb_packets` 跟 `size` 不是一回事，`size` 是从 `packet->size` 获得的字节数。你可以看到我们这里还有 `SDL_mutex` 和 `SDL_cond` 成员，这是因为 SDL 是在一个独立的线程里面来处理音频，所以我们需要有互斥机制来保证对队列数据的正确操作。

下面是队列初始化的代码：

```
void packet_queue_init(PacketQueue *q) {
	memset(q, 0, sizeof(PacketQueue));
	q->mutex = SDL_CreateMutex();
	q->cond = SDL_CreateCond();
}
```

下面是向队列添加数据的代码：

```
int packet_queue_put(PacketQueue *q, AVPacket *pkt) {
	AVPacketList *pkt1;
	if (av_packet_ref(pkt, pkt) < 0) {
		return -1;
	}
	pkt1 = av_malloc(sizeof(AVPacketList));
	if (!pkt1) {
		return -1;
	}
	pkt1->pkt = *pkt;
	pkt1->next = NULL;
	
	
	SDL_LockMutex(q->mutex);
	
	if (!q->last_pkt) {
		q->first_pkt = pkt1;
	}
	else {
		q->last_pkt->next = pkt1;
	}
	q->last_pkt = pkt1;
	q->nb_packets++;
	q->size += pkt1->pkt.size;
	SDL_CondSignal(q->cond);
	
	SDL_UnlockMutex(q->mutex);
	return 0;
}
```

`SDL_LockMutex()` 通过锁住 `mutex` 来让我们能安全的想队列里写入数据。`SDL_CondSignal()` 则在添加完数据后发出信号告诉数据消费方准备获取数据进行下一步处理，同时 `SDL_UnlockMutex()` 解锁 `mutex` 让消费方能正常获取数据。


下面是对应的从队列取数据的代码：

```
int quit = 0;
static int packet_queue_get(PacketQueue *q, AVPacket *pkt, int block) {
	AVPacketList *pkt1;
	int ret;
	
	SDL_LockMutex(q->mutex);
	
	for (;;) {
		if (quit) {
			ret = -1;
			break;
		}
		
		pkt1 = q->first_pkt;
		if (pkt1) {
			q->first_pkt = pkt1->next;
			if (!q->first_pkt) {
				q->last_pkt = NULL;
			}
			q->nb_packets--;
			q->size -= pkt1->pkt.size;
			*pkt = pkt1->pkt;
			av_free(pkt1);
			ret = 1;
			break;
		} else if (!block) {
			ret = 0;
			break;
		} else {
			SDL_CondWait(q->cond, q->mutex);
		}
	}
	SDL_UnlockMutex(q->mutex);
	return ret;
}
```

在上面代码中，我们实现了一个 for 循环，当这个 for 循环遇到阻塞了那就说明肯定是得到一组数据了。我们通过 SDL 的 `SDL_CondWait()` 函数来避免永远循环。基本上所有的 `SDL_CondWait()` 都会等待 `SDL_CondSignal()` 或者 `SDL_CondBroadcast()` 的信号，然后继续。看起好像我们已经在 mutex 这死锁了，因为如果我们不开锁 `packet_queue_put()` 函数就无法向队列里写数据，但事实上 `SDL_CondWait()` 函数是会在合适的时候解开我们传给它的锁的，并在收到信号时再尝试去锁上。


## 程序退出

代码中我们有一个全局变量 `quit`，这个变量是为了当我们在界面上点了退出后，能告诉线程退出。

```
SDL_PollEvent(&event);
switch (event.type) {
	case SDL_QUIT:
		quit = 1;
		SDL_Quit();
		exit(0);
		break;
	default:
		break;
}
```

## 获取数据包





以上便是我们这节教程的全部代码，你可以从这里获得：[https://github.com/samirchen/TestFFmpeg][6]

## 编译执行

你可以使用下面的命令编译它：

```
$ gcc -o tutorial03 tutorial03.c -lavutil -lavformat -lavcodec -lswscale -lz -lm `sdl-config --cflags --libs`
```

找一个视频文件，你可以这样执行一下试试：

```
$ tutorial03 myvideofile.mp4
```





[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ffmpeg-tutorial-3

