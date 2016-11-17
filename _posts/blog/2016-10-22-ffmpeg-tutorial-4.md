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

简单看看 `VideoState` 中都有什么。首先，格式信息 `pFormatCtx`；视频和音频流的标记 `videoStream`、`audioStream`，以及对应的视频和音频流对象 `audio_st`、`video_st`；接下来，是我们移过来的音频缓冲区相关的数据：`audio_buf`、`audio_buf_size`、`audio_buf_index` 等；我们还添加了视频数据队列 `videoq`、视频数据缓冲区 `pictq` 来存储解码后的视频帧，`VideoPicture` 是我们自己创建的数据结构，我们后面再看里面都有些啥；我们还增加了两个指向对应线程的指针：`parse_tid`、`video_tid`；此外还有退出标志 `quit`，媒体文件名 `filename` 等等。


现在我们回到 main 函数来看看我们的程序都有哪些改变。首先，我们创建 `VideoState` 并分配内存：

```
int main(int argc, char *argv[]) {
	SDL_Event event;
	VideoState *is;
	is = av_mallocz(sizeof(VideoState));
	// ... code ...
}
```

`av_mallocz()` 会为我们分配内存并将内存初始化为 0。

接着，初始化视频渲染相关数据缓冲区 `pictq` 的锁。因为事件循环调用我们的渲染函数时，渲染逻辑就会从 `pictq` 获取数据，同时解码逻辑又会往 `pictq` 写入数据，我们不知道谁会先到，所以这里需要通过锁机制来防止线程错乱。同时，我们这里把媒体文件路径也拷贝到 `VideoState` 中。



```
av_strlcpy(is->filename, argv[1], sizeof(is->filename));

is->pictq_mutex = SDL_CreateMutex();
is->pictq_cond = SDL_CreateCond();
```

`av_strlcpy` 是 FFmpeg 基于 `strncpy` 提供的一个字符串拷贝方法，增加了一些边界检查功能。


## 第一个线程

现在我们启动 `decode_thread()` 线程来开始工作：

```
schedule_refresh(is, 40);
	
is->parse_tid = SDL_CreateThread(decode_thread, is);
if (!is->parse_tid) {
	av_free(is);
	return -1;
}
```

我们将在后面实现 `schedule_refresh()` 函数，它的主要功能就是告诉系统在指定的延时后来推送一个 `FF_REFRESH_EVENT` 事件。这个事件将在事件队列里触发 video refresh 函数的调用。不过，现在我们还是先来看看 `SDL_CreateThread()` 函数。

`SDL_CreateThread()` 函数会分发一个新的线程，这个线程有原进程的所有内存的访问权限，并从我们指定的函数开始运行。这个线程会给指定的函数传入一个用户定义的数据作为参数，在我们这里，我们调用的函数是 `decode_thread()` 传入的参数是前面初始化的 `VideoState`。`decode_thread()` 的前半部分没有什么新鲜的：打开媒体文件找到视频流和音频流的索引。这里与之前唯一的不同就是 `AVFormatContext` 被我们放到 `VideoState` 中去了。在找到了视频流和音频流后，我们接下来就调用另一个我们将实现的函数: `stream_component_open()`。这里我们就将代码模块化了，对重复的工作完成了一些代码复用。


`stream_component_open()` 函数主要用于帮我们找到对应的解码器、创建对应的音频配置、保存关键信息到 `VideoState`、启动音频和视频线程。这个函数也是我们添加其他配置的地方，比如强制使用给定的 codec 而不是自动检测等等。代码如下：


```
int stream_component_open(VideoState *is, int stream_index) {
	
	AVFormatContext *pFormatCtx = is->pFormatCtx;
	AVCodecContext *codecCtx = NULL;
	AVCodec *codec = NULL;
	AVDictionary *optionsDict = NULL;
	SDL_AudioSpec wanted_spec, spec;
	
	if (stream_index < 0 || stream_index >= pFormatCtx->nb_streams) {
		return -1;
	}
	
	// Get a pointer to the codec context for the video stream.
	codecCtx = pFormatCtx->streams[stream_index]->codec;
	
	if (codecCtx->codec_type == AVMEDIA_TYPE_AUDIO) {
		// Set audio settings from codec info.
		wanted_spec.freq = codecCtx->sample_rate;
		wanted_spec.format = AUDIO_S16SYS;
		wanted_spec.channels = codecCtx->channels;
		wanted_spec.silence = 0;
		wanted_spec.samples = SDL_AUDIO_BUFFER_SIZE;
		wanted_spec.callback = audio_callback;
		wanted_spec.userdata = is;
		
		if (SDL_OpenAudio(&wanted_spec, &spec) < 0) {
			fprintf(stderr, "SDL_OpenAudio: %s\n", SDL_GetError());
			return -1;
		}
	}
	codec = avcodec_find_decoder(codecCtx->codec_id);
	if (!codec || (avcodec_open2(codecCtx, codec, &optionsDict) < 0)) {
		fprintf(stderr, "Unsupported codec!\n");
		return -1;
	}
	
	switch(codecCtx->codec_type) {
		case AVMEDIA_TYPE_AUDIO:
			is->audioStream = stream_index;
			is->audio_st = pFormatCtx->streams[stream_index];
			is->audio_buf_size = 0;
			is->audio_buf_index = 0;
			memset(&is->audio_pkt, 0, sizeof(is->audio_pkt));
			packet_queue_init(&is->audioq);
			SDL_PauseAudio(0);
			break;
		case AVMEDIA_TYPE_VIDEO:
			is->videoStream = stream_index;
			is->video_st = pFormatCtx->streams[stream_index];
			
			packet_queue_init(&is->videoq);
			is->video_tid = SDL_CreateThread(video_thread, is);
			is->sws_ctx = sws_getContext(is->video_st->codec->width, is->video_st->codec->height, is->video_st->codec->pix_fmt, is->video_st->codec->width, is->video_st->codec->height, AV_PIX_FMT_YUV420P, SWS_BILINEAR, NULL, NULL, NULL);
			break;
		default:
			break;
	}
	return 0;
}
```

上面的函数主要是服务于音频和视频，这里把 `VideoState` 作为回调函数的参数数据。同时，我们保存了 `audio_st` 和 `video_st`，还初始化了视频队列 `videoq` 和音频队列 `audioq`。最重要的是，我们在这里启动了音频线程和视频线程。


```
SDL_PauseAudio(0);
break;

// ...... 

is->video_tid = SDL_CreateThread(video_thread, is);
```


我们接着看看 `decode_thread()` 的后半部分，这部分的主要工作是通过一个循环来读取 packet 并把它放入正确的队列：


```
int decode_thread(void *arg) {

	// ... code ...

	// Main decode loop.
	for (;;) {
		if (is->quit) {
			break;
		}
		// Seek stuff goes here.
		if (is->audioq.size > MAX_AUDIOQ_SIZE || is->videoq.size > MAX_VIDEOQ_SIZE) {
			SDL_Delay(10);
			continue;
		}
		if (av_read_frame(is->pFormatCtx, packet) < 0) {
			if (is->pFormatCtx->pb->error == 0) {
				SDL_Delay(100); // No error; wait for user input.
				continue;
			} else {
				break;
			}
		}
		// Is this a packet from the video stream?
		if (packet->stream_index == is->videoStream) {
			packet_queue_put(&is->videoq, packet);
		} else if (packet->stream_index == is->audioStream) {
			packet_queue_put(&is->audioq, packet);
		} else {
			av_packet_unref(packet);
		}
	}

	// All done - wait for it.
	while (!is->quit) {
		SDL_Delay(100);
	}

	fail:
	if (1) {
		SDL_Event event;
		event.type = FF_QUIT_EVENT;
		event.user.data1 = is;
		SDL_PushEvent(&event);
	}
	return 0;
}
```


上面代码的 for 循环中，我们为音频队列和视频队列添加了 max size，还增加了对读数据错误的检查。`AVFormatContext *pFormatCtx` 有一个 `ByteIOContext` 成员，这个成员会记录所有底层文件信息。

在循环完成后，接下来的逻辑就是等待其他任务结束，以及发出通知告诉其他任务我们这已经结束了。这段扫尾代码也演示了如何发事件。

我们通过 SDL 提供的常量 `SDL_USEREVENT` 来取得用户事件，第一个用户事件的值为 `SDL_USEREVENT`，往后则都是累加 1。比如，`FF_QUIT_EVENT` 事件在我们的程序中的值是 `SDL_USEREVENT + 1`。我们还可以给事件附加上用户数据，我们的程序中，我们把用户数据的指针指向了 `is`。最后我们调用 `SDL_PushEvent()` 函数将事件发布出去。在后续的事件处理逻辑中，我们将遍历和处理事件。现在要明确的就是我们这里发出了 `FF_QUIT_EVENT` 事件，我们将获取这个事件并将 `quit` 标志置为 1。



## 获取帧：video_thread

在 codec 准备好后，我们启动 video thread。这个线程从 video queue 中读取数据包 packet，解码为视频帧，然后调用 `queue_picture()` 函数将处理好的帧添加到 picture queue。


```
int video_thread(void *arg) {
	VideoState *is = (VideoState *) arg;
	AVPacket pkt1, *packet = &pkt1;
	int frameFinished;
	AVFrame *pFrame;
	
	pFrame = av_frame_alloc();
	
	for (;;) {
		if (packet_queue_get(&is->videoq, packet, 1) < 0) {
			// Means we quit getting packets.
			break;
		}
		// Decode video frame.
		avcodec_decode_video2(is->video_st->codec, pFrame, &frameFinished, packet);
		
		// Did we get a video frame?
		if (frameFinished) {
			if (queue_picture(is, pFrame) < 0) {
				break;
			}
		}
		av_packet_unref(packet);
	}
	av_free(pFrame);
	return 0;
}
```

这里的代码还是比较清晰的，我们把 `avcodec_decode_video2()` 函数挪到了这里，由于很多信息被我们放到了 `VideoState` 中，所以这里的参数我们做了改变，比如：我们通过 `is->video_st->codec` 从 `VideoState` 中获取视频的 codec。我们持续从 video queue 中获取 packet 数据包，直到有人告诉我们 quit 或者遇到错误。


## 帧队列

接着，我们看一下存储解码帧的函数 `queue_picture()`，由于我们的 picture queue 里放的是 SDL overlay，所以我们需要把视频帧转换为 SDL overlay。


```
typedef struct VideoPicture {
	SDL_Overlay *bmp;
	int width, height; // Source height & width..
	int allocated;
} VideoPicture;
```


`VideoState` 中有个缓冲区用来存储 `VideoPicture`，但是我们需要自己创建和分配 `SDL_Overlay` 的内存，注意，`allocated` 就是用来标记我们有没有做这件事。


我们需要两个指针来帮助我们使用这个队列：写索引和读索引。我们同时也记录缓冲区有多少图像。当要往队列写入数据时，我们首先要等缓冲区清理出空间来存放 `VideoPicture`。然后我们检查我们是否在写索引位置创建了 SDL overlay，如果没有则需要分配对应的内存。如果窗口的尺寸发生改变了，我们还要重新创建缓冲区。


```
int queue_picture(VideoState *is, AVFrame *pFrame) {
	
	VideoPicture *vp;
	AVFrame pict;
	
	// Wait until we have space for a new pic.
	SDL_LockMutex(is->pictq_mutex);
	while (is->pictq_size >= VIDEO_PICTURE_QUEUE_SIZE && !is->quit) {
		SDL_CondWait(is->pictq_cond, is->pictq_mutex);
	}
	SDL_UnlockMutex(is->pictq_mutex);
	
	if (is->quit) {
		return -1;
	}
	
	// windex is set to 0 initially.
	vp = &is->pictq[is->pictq_windex];
	
	// Allocate or resize the buffer!
	if (!vp->bmp || vp->width != is->video_st->codec->width || vp->height != is->video_st->codec->height) {
		SDL_Event event;
		
		vp->allocated = 0;
		// We have to do it in the main thread.
		event.type = FF_ALLOC_EVENT;
		event.user.data1 = is;
		SDL_PushEvent(&event);
		
		// Wait until we have a picture allocated.
		SDL_LockMutex(is->pictq_mutex);
		while (!vp->allocated && !is->quit) {
			SDL_CondWait(is->pictq_cond, is->pictq_mutex);
		}
		SDL_UnlockMutex(is->pictq_mutex);
		if (is->quit) {
			return -1;
		}
	}
	
	// ... code ...

}
```

这里我们发送了一个事件 `FF_ALLOC_EVENT`，处理这个事件的代码在主线程中：

```
case FF_ALLOC_EVENT:
	alloc_picture(event.user.data1);
	break;
```

这里调用了 `alloc_picture()` 函数，我们来看一下这个函数：

```
void alloc_picture(void *userdata) {
	
	VideoState *is = (VideoState *)userdata;
	VideoPicture *vp;
	
	vp = &is->pictq[is->pictq_windex];
	if (vp->bmp) {
		// We already have one make another, bigger/smaller.
		SDL_FreeYUVOverlay(vp->bmp);
	}
	// Allocate a place to put our YUV image on that screen.
	SDL_LockMutex(screen_mutex);
	vp->bmp = SDL_CreateYUVOverlay(is->video_st->codec->width, is->video_st->codec->height, SDL_YV12_OVERLAY, screen);
	SDL_UnlockMutex(screen_mutex);
	vp->width = is->video_st->codec->width;
	vp->height = is->video_st->codec->height;
	
	SDL_LockMutex(is->pictq_mutex);
	vp->allocated = 1;
	SDL_CondSignal(is->pictq_cond);
	SDL_UnlockMutex(is->pictq_mutex);
	
}
```

我们把 `SDL_CreateYUVOverlay()` 从 main 函数中移到了这里，现在我们对这个函数加了锁，因为有两个线程可以同时往屏幕写数据，这样能防止 `alloc_picture()` 函数和显示视频的函数发生冲突。需要注意我们在 `VideoPicture` 中记录了视频的宽度和高度，因为我们需要确保我们的视频尺寸不会发生改变。


现在我们已经创建了 YUV overlay 并分配了内存，我们做好了接收图像的准备。现在我们回到 `queue_picture()` 函数来看看拷贝视频帧到 YUV overlay 的这部分代码：


```
int queue_picture(VideoState *is, AVFrame *pFrame) {

	// Allocate a frame if we need it... 
	// ... code ...
	// We have a place to put our picture on the queue 

	// We have a place to put our picture on the queue.
	if (vp->bmp) {
		
		SDL_LockYUVOverlay(vp->bmp);
		
		// Point pict at the queue.
		pict.data[0] = vp->bmp->pixels[0];
		pict.data[1] = vp->bmp->pixels[2];
		pict.data[2] = vp->bmp->pixels[1];
		
		pict.linesize[0] = vp->bmp->pitches[0];
		pict.linesize[1] = vp->bmp->pitches[2];
		pict.linesize[2] = vp->bmp->pitches[1];
		
		// Convert the image into YUV format that SDL uses.
		sws_scale(is->sws_ctx, (uint8_t const * const *)pFrame->data, pFrame->linesize, 0, is->video_st->codec->height, pict.data, pict.linesize);
		
		SDL_UnlockYUVOverlay(vp->bmp);
		// Now we inform our display thread that we have a pic ready.
		if (++is->pictq_windex == VIDEO_PICTURE_QUEUE_SIZE) {
			is->pictq_windex = 0;
		}
		SDL_LockMutex(is->pictq_mutex);
		is->pictq_size++;
		SDL_UnlockMutex(is->pictq_mutex);
	}
	return 0;
}
```

这段代码主要是用视频帧来填充 YUV overlay 的逻辑。最后一段代码就是向队列添加数据，这个队列工作的方式就是不断地往里添加数据直到队列满掉，然后不断从中读取数据只要里面还有数据，所以读写操作都会依赖 `is->pictq_size` 的值，所以这里我们要给它加锁。我们在这里做的就是，将写指针递增，然后锁住队列并增加它的 size。然后读数据方会知道队列中有数据了，如果队列满了，我们写数据方也会知道。





## 显示视频

上面介绍完了 video thread，我们接下来来看看 `schedule_refresh()`：


```
// Schedule a video refresh in 'delay' ms.
static void schedule_refresh(VideoState *is, int delay) {
	SDL_AddTimer(delay, sdl_refresh_timer_cb, is);
}
```

`SDL_AddTimer()` 是一个 SDL 的函数，用来在指定的时间(ms)后回调用户指定的函数，当然还可以选择带上用户指定的数据。我们将用 `schedule_refresh` 这个函数来做图像更新：每次我们调用这个函数，它就会设置一个定时器，这个定时器会触发一个事件来让 main 函数的事件处理逻辑去从 picture queue 取得一帧数据来显示出来。


```
static Uint32 sdl_refresh_timer_cb(Uint32 interval, void *opaque) {
	SDL_Event event;
	event.type = FF_REFRESH_EVENT;
	event.user.data1 = opaque;
	SDL_PushEvent(&event);
	return 0; // 0 means stop timer.
}
```

`sdl_refresh_timer_cb()` 就是定时器会触发调用的那个用于发事件的函数，`FF_REFRESH_EVENT` 事件被定义为了 `SDL_USEREVENT + 1`。主要注意的是当我们在这里返回 0 时，SDL 会停止这个定时器，这样也就停止去调用这个回调了。


既然我们这里发出了 `FF_REFRESH_EVENT` 事件，那么就需要有地方处理它，这个地方就在 main 函数的 event loop 中：

```
for (;;) {
	
	SDL_WaitEvent(&event);
	switch(event.type) {

		// ... code ...
		
		case FF_REFRESH_EVENT:
			video_refresh_timer(event.user.data1);
			break;

		// ... code ...

	}
}
```

从这里可以看到，处理这个事件的函数是 `video_refresh_timer()`：


```
void video_refresh_timer(void *userdata) {
	
	VideoState *is = (VideoState *)userdata;
	// vp is used in later tutorials for synchronization.
	VideoPicture *vp;
	
	if (is->video_st) {
		if (is->pictq_size == 0) {
			schedule_refresh(is, 1);
		} else {
			vp = &is->pictq[is->pictq_rindex];

			// Now, normally here goes a ton of code about timing, etc. we're just going to guess at a delay for now. You can increase and decrease this value and hard code the timing - but I don't suggest that, We'll learn how to do it for real later.
			schedule_refresh(is, 80);
			
			// Show the picture!
			video_display(is);
			
			// Update queue for next picture!
			if (++is->pictq_rindex == VIDEO_PICTURE_QUEUE_SIZE) {
				is->pictq_rindex = 0;
			}
			SDL_LockMutex(is->pictq_mutex);
			is->pictq_size--;
			SDL_CondSignal(is->pictq_cond);
			SDL_UnlockMutex(is->pictq_mutex);
		}
	} else {
		schedule_refresh(is, 100);
	}
}
```

这是一个比较简单的函数：当 `pictq` 队列有数据时就取出 `VideoPicture`，设置显示下一帧图像的 timer，调用 `video_display()` 来将视频显示出来，增加队列的计数器，更新队列的 size。你可能注意到了，这里我们虽然取出了一个 `VideoPicture` 但并没有使用它，原因是我们后面会用到。后面我们会用这个 `VideoPicture` 的时间信息来做音视频同步相关的工作，其中的时间信息将告诉我们该何时显示下一帧图像，我们会把这个时间信息传给 `schedule_refresh()`。而现在，我们只是简单的传了一个 80。


现在我们要做的最后一件事就是 `video_display()` 函数：

```
void video_display(VideoState *is) {
	SDL_Rect rect;
	VideoPicture *vp;
	float aspect_ratio;
	int w, h, x, y;
	
	vp = &is->pictq[is->pictq_rindex];
	if (vp->bmp) {
		if (is->video_st->codec->sample_aspect_ratio.num == 0) {
			aspect_ratio = 0;
		} else {
			aspect_ratio = av_q2d(is->video_st->codec->sample_aspect_ratio) * is->video_st->codec->width / is->video_st->codec->height;
		}
		if (aspect_ratio <= 0.0) {
			aspect_ratio = (float) is->video_st->codec->width / (float) is->video_st->codec->height;
		}
		h = screen->h;
		w = ((int)rint(h * aspect_ratio)) & -3;
		if (w > screen->w) {
			w = screen->w;
			h = ((int)rint(w / aspect_ratio)) & -3;
		}
		x = (screen->w - w) / 2;
		y = (screen->h - h) / 2;
		
		rect.x = x;
		rect.y = y;
		rect.w = w;
		rect.h = h;
		SDL_LockMutex(screen_mutex);
		SDL_DisplayYUVOverlay(vp->bmp, &rect);
		SDL_UnlockMutex(screen_mutex);
	}
}
```

由于我们的屏幕可以是任意尺寸（我们自己设置的是 640x480，但是这个对用户应该是可以改变的），所以我们需要能够动态地计算我们要显示图像的尺寸。首先，我们需要计算出视频的 **aspect ratio**，即宽度和高度的比例(width/height)。但是有一些 codec 有很奇怪的 **sample aspect ration**，即单像素(单采样)的宽高比(width/height)，又由于我们的 `AVCodecContext` 中的宽度和高度是以像素为单位来表示的，那么这时候 actual aspect ratio 应该是 aspect ratio 乘上 sample aspect ratio。有的 codec 的 aspect ratio 值是 0，这表示的是每个像素的尺寸是 1x1。

接下来，我们放大视频来尽量适配我们的屏幕。代码中的 `& -3` 位操作可以将数值调整到最接近 4 的倍数，然后我们将视频居中，并调用 `SDL_DisplayYUVOverlay()`，这里要确保我们通过 `screen_mutex` 来加锁。


到这里，我们还需要用新的 `VideoState` 来重写音频处理的代码，但这里的工作还是比较少的，参见样例代码。最后我们要修改一下 FFmpeg 的内部退出回调对应的函数：


```
// Since we only have one decoding thread, the Big Struct can be global in case we need it.
VideoState *global_video_state;

int decode_interrupt_cb(void *opaque) {
	return (global_video_state && global_video_state->quit);
}
```

我们在主函数设置 `global_video_state` 为 `VideoState *is`。



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
[3]: http://dranger.com/ffmpeg/tutorial04.html