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

## 填充队列数据

接着要做的就是创建我们的队列：

```
PacketQueue audioq;
int main(int argc, char *argv[]) {
	// ... code ...

	avcodec_open2(aCodecCtx, aCodec, &audioOptionsDict);

	// audio_st = pFormatCtx->streams[index].
	packet_queue_init(&audioq);
	SDL_PauseAudio(0);
	
	// ... code ...
}
```

`SDL_PauseAudio()` 最终开启了音频设备，如果这时候没有获得数据，那么它就静音。

一旦当我们的队列建立起来，我们就可以开始往里面填充数据包了。下面就是我们用于填数据的循环：

```
while (av_read_frame(pFormatCtx, &packet) >= 0) {
	// Is this a packet from the video stream?
	if (packet.stream_index == videoStream) {
		// Decode video frame.
		
		// ... code ...

	} else if (packet.stream_index == audioStream) {
		packet_queue_put(&audioq, &packet);
	} else {
		// Free the packet that was allocated by av_read_frame.
		av_packet_unref(&packet);
	}

	// ... code ...

}
```

注意，我们没有在把音频数据包 packet 放入队列后就立即释放它，我们会在解码它之后才释放。


## 获取队列数据


这里我们开始实现我们获取和处理队列数据的回调函数，这个回调函数必须遵循这样的形式：`void callback(void *userdata, Uint8 *stream, int len)`，其中 `userdata` 是我们给 SDL 的指针，`stream` 是我们写入音频数据的缓冲区，`len` 是缓冲区的大小。


```
void audio_callback(void *userdata, Uint8 *stream, int len) {
	AVCodecContext *aCodecCtx = (AVCodecContext *)userdata;
	int len1, audio_size;
	
	static uint8_t audio_buf[(MAX_AUDIO_FRAME_SIZE * 3) / 2];
	static unsigned int audio_buf_size = 0;
	static unsigned int audio_buf_index = 0;
	
	while (len > 0) {
		if (audio_buf_index >= audio_buf_size) {
			// We have already sent all our data; get more.
			audio_size = audio_decode_frame(aCodecCtx, audio_buf, audio_buf_size);
			if (audio_size < 0) {
				// If error, output silence.
				audio_buf_size = 1024; // arbitrary?
				memset(audio_buf, 0, audio_buf_size);
			} else {
				audio_buf_size = audio_size;
			}
			audio_buf_index = 0;
		}
		len1 = audio_buf_size - audio_buf_index;
		if (len1 > len) {
			len1 = len;
		}
		memcpy(stream, (uint8_t *) audio_buf + audio_buf_index, len1);
		len -= len1;
		stream += len1;
		audio_buf_index += len1;
	}
}
```

这里实现了一个简单的循环来拉取数据，`audio_decode_frame()` 会存储解码结果在一个临时缓冲区，这个缓冲区的数据会流向 `stream`。`audio_buf` 这个缓冲区的大小是 FFmpeg 给我们的最大音频帧大小的 1.5 倍，从而起到一个很好的弹性的作用。


## 音频解码

音频解码的代码都在 `audio_decode_frame()` 函数中实现：

```
int audio_decode_frame(AVCodecContext *aCodecCtx, uint8_t *audio_buf, int buf_size) {
	static AVPacket pkt;
	static uint8_t *audio_pkt_data = NULL;
	static int audio_pkt_size = 0;
	static AVFrame frame;
	
	int len1, data_size = 0;
	
	for (;;) {
		while(audio_pkt_size > 0) {
			int got_frame = 0;
			len1 = avcodec_decode_audio4(aCodecCtx, &frame, &got_frame, &pkt);
			if (len1 < 0) {
				// if error, skip frame.
				audio_pkt_size = 0;
				break;
			}
			audio_pkt_data += len1;
			audio_pkt_size -= len1;
			if (got_frame) {
				data_size = av_samples_get_buffer_size(NULL, aCodecCtx->channels, frame.nb_samples, aCodecCtx->sample_fmt, 1);
				memcpy(audio_buf, frame.data[0], data_size);
			}
			if (data_size <= 0) {
				// No data yet, get more frames.
				continue;
			}
			// We have data, return it and come back for more later.
			return data_size;
		}
		if (pkt.data) {
			av_packet_unref(&pkt);
		}
		
		if (quit) {
			return -1;
		}
		
		if (packet_queue_get(&audioq, &pkt, 1) < 0) {
			return -1;
		}
		audio_pkt_data = pkt.data;
		audio_pkt_size = pkt.size;
	}
}
```

我们从最后面开始看这整段代码的处理逻辑，我们调用 `packet_queue_get()` 函数从队列中取数据包并存下来，接着一旦我们有了数据包就调用 `avcodec_decode_audio4()` 函数来进行解码。在一些情况下，一个数据包 packet 可能有超过 1 个帧，那样就需要多次调用这段处理逻辑来从数据包里取得所有数据。一旦我们取得了一个帧，我们就把它拷贝的音频缓冲区。这里需要注意的是数据类型转换，因为 SDL 给我们的是 8 bit 的整型缓冲区，但是 FFmpeg 给我们的数据是 16 bit 的整型缓冲区。另外，还需要搞清楚 `len1` 和 `data_size` 的区别，`len1` 是一个 packet 中已被我们使用的字节数，`data_size` 是返回的原生数据的大小。


当我们获取到一些数据后就立即返回来看看是否需要从队列中获得更多的数据或者已经可以足够。如果一个 packet 中还有更多的数据需要继续处理，我们就将这些数据保存一会；如果已经处理完一个 packet 中的所有数据，那我们就释放这个 packet 的内存。

略作总结，主线程的数据读取的循环负责从媒体文件中读取数据并写入到队列，我们从队列中取出数据给 `audio_callback` 回调函数处理，回调函数会将数据交给 SDL，SDL 将数据交给声卡来播放出声音。



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
[3]: http://dranger.com/ffmpeg/tutorial03.html
