---
layout: post
title: FFmpeg 入门(5)：视频同步
description: FFmpeg 入门教程系列。
category: blog
tag: FFmpeg, tutorial
---

## 视频如何同步

在之前的教程中，我们已经可以开始播放视频了，也已经可以开始播放音频了，但是视频和音频的播放还未同步，我们要怎么办呢？

### PTS 和 DTS 

好在音频和视频都有信息来控制播放时的速度和时机。音频流有一个**采样率(sample rate)**，视频流有一个**帧率(frame per second)**。但是，如果我们只是简单地通过数帧和乘上帧率来同步视频，那么它可能会和音频不同步。实际上我们将使用 PTS 和 DTS 信息来做音视频同步相关的事情。

在介绍 PTS 和 DTS 的概念前，先来了解一下 I、P、B 帧的概念。视频的播放过程可以简单理解为一帧一帧的画面按照时间顺序呈现出来的过程，就像在一个本子的每一页画上画，然后快速翻动的感觉。但是在实际应用中，并不是每一帧都是完整的画面，因为如果每一帧画面都是完整的图片，那么一个视频的体积就会很大，这样对于网络传输或者视频数据存储来说成本太高，所以通常会对视频流中的一部分画面进行压缩（编码）处理。由于压缩处理的方式不同，视频中的画面帧就分为了不同的类别，其中包括：I 帧、P 帧、B 帧。

I 帧、P 帧、B 帧的区别在于：

- I 帧（Intra coded frames）：I 帧图像采用帧内编码方式，即只利用了单帧图像内的空间相关性，而没有利用时间相关性。I 帧使用帧内压缩，不使用运动补偿，由于 I 帧不依赖其它帧，所以是随机存取的入点，同时是解码的基准帧。I 帧主要用于接收机的初始化和信道的获取，以及节目的切换和插入，I 帧图像的压缩倍数相对较低。I 帧图像是周期性出现在图像序列中的，出现频率可由编码器选择。
- P 帧（Predicted frames）：P 帧和 B 帧图像采用帧间编码方式，即同时利用了空间和时间上的相关性。P 帧图像只采用前向时间预测，可以提高压缩效率和图像质量。P 帧图像中可以包含帧内编码的部分，即 P 帧中的每一个宏块可以是前向预测，也可以是帧内编码。
- B 帧（Bi-directional predicted frames）：B 帧图像采用双向时间预测，可以大大提高压缩倍数。值得注意的是，由于 B 帧图像采用了未来帧作为参考，因此 MPEG-2 编码码流中图像帧的传输顺序和显示顺序是不同的。

也就是说，一个 I 帧可以不依赖其他帧就解码出一幅完整的图像，而 P 帧、B 帧不行。P 帧需要依赖视频流中排在它前面的帧才能解码出图像。B 帧则需要依赖视频流中排在它前面或后面的帧才能解码出图像。这也解释了为什么当我们调用 `avcodec_decode_video2()` 函数后我们不一定能得到一个完成解码的帧。

这就带来一个问题：在视频流中，先到来的 B 帧无法立即解码，需要等待它依赖的后面的 I、P 帧先解码完成，这样一来播放时间与解码时间不一致了，顺序打乱了，那这些帧该如何播放呢？这时就需要 DTS 和 PTS 信息了。

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


当我们在程序中调用 `av_read_frame()` 函数得到一个 packet 后，它会包含 PTS 和 DTS 信息。但是我们真正想要的是最新解码好的原始帧的 PTS，这个我们才知道什么时候显示这一帧。


### 同步


现在，假设我们现在要显示某一帧视频，我们具体怎么操作呢？现在有一个方案：当我们显示完一帧，我们需要计算什么时候显示下一帧。然后我们设置一个新的定时来在这之后刷新视频。如你所想，我们通过检查下一帧的 PTS 来决定这里的定时是多久。这个方案几乎是可行的，但有两个需要解决的问题：


第一个问题是怎么知道下一帧的 PTS。你可能会想，就在当前的帧的 PTS 上加一个根据视频帧率计算出的时间增量，但是有些视频需要重复帧，那就意味着这种情况下需要重复显示一帧多次，这时候这里说的这个策略就会导致我们提前显示了下一帧。所以我们得考虑一下。

第二个问题是在一切都完美的情况下，音视频都按照正确的节奏播放，这时候我们不会有同步的问题。但是事实上，用户的设备、网络，甚至视频文件都是有可能出现问题的，这时候我们可能要做出选择了：音频同步视频时间、视频同步音频时间、音频和视频同步外部时钟。我们的选择是**视频同步音频时间**。



## 获取帧的 PTS


现在我们把上面的策略实现到代码里。我们需要在 `VideoState` 结构体中再增加一些成员。我们再来看看 video thread，这里是我们从队列获取 packets 的地方，这些 packets 是 decode thread 放入的。我们要做的是当调用 `avcodec_decode_video2` 获得 frame 时，计算 PTS 数据。


```
AVFrame *pFrame;
double pts;

pFrame = av_frame_alloc();

for (;;) {
	if (packet_queue_get(&is->videoq, packet, 1) < 0) {
		// Means we quit getting packets.
		break;
	}
	pts = 0;
	
	// Save global pts to be stored in pFrame in first call.
	global_video_pkt_pts = packet->pts;
	// Decode video frame.
	avcodec_decode_video2(is->video_st->codec, pFrame, &frameFinished, packet);
	if (packet->dts == AV_NOPTS_VALUE && pFrame->opaque && *(uint64_t*)pFrame->opaque != AV_NOPTS_VALUE) {
		pts = *(uint64_t *)pFrame->opaque;
	} else if (packet->dts != AV_NOPTS_VALUE) {
		pts = packet->dts;
	} else {
		pts = 0;
	}
	pts *= av_q2d(is->video_st->time_base);

	// ... code ...

}
```

当我们无法计算 PTS 时就设置它为 0。


一个需要注意的地方，我们在这里使用了 `int64` 来存储 PTS，这是因为 PTS 是一个整型值。比如，如果一个视频流的帧率是 24，那么 PTS 为 42 则表示这一帧应该是第 42 帧如果我们 1/24 秒播一帧的话。我们可以用这个值除以帧率来得到以秒为单位的时间。视频流的 `time_base` 值则是 1/framerate，所以当我们获得 PTS 后，我们要乘上 `time_base`。



## 用 PTS 来同步

现在 PTS 值已经被算出来了，那么接下来我们来处理上面说到的两个同步问题。我们将定义一个函数 `synchronize_video()` 来用于更新需要同步的视频帧的 PTS。这个函数同时也会处理没有获得 PTS 的情况。同时，我们还要跟踪何时需要下一帧以便于我们设置合理的刷新率。我们可以使用一个内置的 `video_clock` 变量来跟踪视频已经播过的时间。我们把这个变量加到了 `VideoState` 中。



```
typedef struct VideoState {
	// ... code ...
	double video_clock; // pts of last decoded frame / predicted pts of next decoded frame.
	// ... code ...
}

double synchronize_video(VideoState *is, AVFrame *src_frame, double pts) {
	double frame_delay;
	
	if (pts != 0) {
		// If we have pts, set video clock to it.
		is->video_clock = pts;
	} else {
		// If we aren't given a pts, set it to the clock.
		pts = is->video_clock;
	}
	// Update the video clock.
	frame_delay = av_q2d(is->video_st->codec->time_base);
	// If we are repeating a frame, adjust clock accordingly.
	frame_delay += src_frame->repeat_pict * (frame_delay * 0.5);
	is->video_clock += frame_delay;
	return pts;
}
```

你可以看到这个函数也同时处理了帧重复的情况。

接下来，我们给 `queue_picture` 加了个 `pts` 参数，在调用 `synchronize_video` 获取同步的 PTS 后，把这个值传入：

```
// Did we get a video frame?
if (frameFinished) {
	pts = synchronize_video(is, pFrame, pts);
	if (queue_picture(is, pFrame, pts) < 0) {
		break;
	}
}
```

同时我们还更新了 `VideoPicture` 这个数据结构，添加了 `pts` 成员：

```
typedef struct VideoPicture {
	// ... code ...
	double pts;
} VideoPicture;
```

这样在 `queue_picture` 这里的变化即增加了一行保存 `pts` 值到 `VideoPicture` 的代码：

```
int queue_picture(VideoState *is, AVFrame *pFrame, double pts) {

	// ... code ...

	if (vp->bmp) {
		// ... covert picture ...

		vp->pts = pts;

		// ... alert queue ...
	}

	return 0;
}

```

所以现在我们的 picture queue 中等待显示的图像都是有着合适的 PTS 值的了。现在让我们来看看 `video_refresh_timer()` 这个用了刷新视频显式的函数。在上一节我们简单的设置了一下刷新时间间隔为 80ms，现在我们要根据 PTS 来计算它。



```
void video_refresh_timer(void *userdata) {
	VideoState *is = (VideoState *) userdata;
	VideoPicture *vp;
	double actual_delay, delay, sync_threshold, ref_clock, diff;
	
	if (is->video_st) {
		if (is->pictq_size == 0) {
			schedule_refresh(is, 1);
		} else {
			vp = &is->pictq[is->pictq_rindex];
			
			delay = vp->pts - is->frame_last_pts; // The pts from last time.
			if (delay <= 0 || delay >= 1.0) {
				// If incorrect delay, use previous one.
				delay = is->frame_last_delay;
			}
			// Save for next time.
			is->frame_last_delay = delay;
			is->frame_last_pts = vp->pts;
			
			// Update delay to sync to audio.
			ref_clock = get_audio_clock(is);
			diff = vp->pts - ref_clock;
			
			// Skip or repeat the frame. Take delay into account FFPlay still doesn't "know if this is the best guess."
			sync_threshold = (delay > AV_SYNC_THRESHOLD) ? delay : AV_SYNC_THRESHOLD;
			if (fabs(diff) < AV_NOSYNC_THRESHOLD) {
				if (diff <= -sync_threshold) {
					delay = 0;
				} else if (diff >= sync_threshold) {
					delay = 2 * delay;
				}
			}
			is->frame_timer += delay;
			// Computer the REAL delay.
			actual_delay = is->frame_timer - (av_gettime() / 1000000.0);
			if (actual_delay < 0.010) {
				// Really it should skip the picture instead.
				actual_delay = 0.010;
			}
			schedule_refresh(is, (int)(actual_delay * 1000 + 0.5));
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

我们的策略是通过比较前一个 PTS 和当前的 PTS 来预测下一帧的 PTS。与此同时，我们需要同步视频到音频。我们将创建一个 audio clock 作为内部变量来跟踪音频现在播放的时间点，video thread 将用这个值来计算和判断视频是播快了还是播慢了。


现在假设我们有一个 `get_audio_clock` 函数来返回我们 audio clock，那当我们拿到这个值，我们怎么去处理音视频不同步的情况呢？如果只是简单的尝试跳到正确的 packet 来解决并不是一个很好的方案。我们要做的是调整下一次刷新的时机：如果视频播慢了我们就加快刷新，如果视频播快了我们就减慢刷新。既然我们调整好了刷新时间，接下来用 `frame_timer` 跟电脑的时钟做一下比较。`frame_timer` 会一直累加在播放过程中我们计算的延时。换而言之，这个 `frame_timer` 就是播放下一帧的应该对上的时间点。我们简单的在 `frame_timer` 上累加新计算的 delay，然后和电脑的时钟比较，并用得到的值来作为时间间隔去刷新。这段逻辑需要好好阅读一下下面的代码：



```
void video_refresh_timer(void *userdata) {
	VideoState *is = (VideoState *) userdata;
	VideoPicture *vp;
	double actual_delay, delay, sync_threshold, ref_clock, diff;
	
	if (is->video_st) {
		if (is->pictq_size == 0) {
			schedule_refresh(is, 1);
		} else {
			vp = &is->pictq[is->pictq_rindex];
			
			delay = vp->pts - is->frame_last_pts; // The pts from last time.
			if (delay <= 0 || delay >= 1.0) {
				// If incorrect delay, use previous one.
				delay = is->frame_last_delay;
			}
			// Save for next time.
			is->frame_last_delay = delay;
			is->frame_last_pts = vp->pts;
			
			// Update delay to sync to audio.
			ref_clock = get_audio_clock(is);
			diff = vp->pts - ref_clock;
			
			// Skip or repeat the frame. Take delay into account FFPlay still doesn't "know if this is the best guess."
			sync_threshold = (delay > AV_SYNC_THRESHOLD) ? delay : AV_SYNC_THRESHOLD;
			if (fabs(diff) < AV_NOSYNC_THRESHOLD) {
				if (diff <= -sync_threshold) {
					delay = 0;
				} else if (diff >= sync_threshold) {
					delay = 2 * delay;
				}
			}
			is->frame_timer += delay;
			// Computer the REAL delay.
			actual_delay = is->frame_timer - (av_gettime() / 1000000.0);
			if (actual_delay < 0.010) {
				// Really it should skip the picture instead.
				actual_delay = 0.010;
			}
			schedule_refresh(is, (int)(actual_delay * 1000 + 0.5));
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

有一些需要注意的点：首先，要确保前一个 PTS 以及当前 PTS 和前一个 PTS 间的 delay 是有效值，如果不是，那么我们就用上一个 delay 值。其次，要有一个同步时间戳的阈值，因为我们不可能完美的做到同步，FFPlay 中是用 0.01 作为这个阈值的，我们还要确保这个阈值不要大于两个 PTS 的差值。最后，我们设置最小的刷新时间为 10ms。


我们在 `VideoState` 里加了不少成员，注意检查一下。另外，不要忘了在 `stream_component_open()` 初始化 `frame_timer` 和 `frame_last_delay`。

```
is->frame_timer = (double) av_gettime() / 1000000.0;
is->frame_last_delay = 40e-3;
```


## 音频时钟

现在是时候来实现音频时钟了。我们可以在 `audio_decode_frame()` 中更新音频时钟，这里是音频解码的地方。要记住的是并不是每次调用这个函数时都会处理一个新的 packet，所以有两个地方需要更新时钟：第一个地方是获得一个新的 packet 的时候，这时候设置音频时钟为 packet 的 PTS 即可；如果一个 packet 包含多个 frame 时，我们就通过用播放的音频采样乘上采样率来跟踪音频播放的时间。

获得新 packet 的时候：

```
// If update, update the audio clock w/pts.
if (pkt->pts != AV_NOPTS_VALUE) {
	is->audio_clock = av_q2d(is->audio_st->time_base) * pkt->pts;
}
``
一个 packet 包含多个 frame 的时候：

```
pts = is->audio_clock;
*pts_ptr = pts;
n = 2 * is->audio_st->codec->channels;
is->audio_clock += (double) data_size / (double) (n * is->audio_st->codec->sample_rate);
```			

一些细节：`audio_decode_frame` 函数添加了一个 `pts_ptr` 参数，它是一个指针，我们用它来告知 `audio_callback()` 音频的 packet。这个会在后面同步音频和视频时起到作用。


最后我们来实现 `get_audio_clock()` 函数。这里不是简单的获得 `is->audio_clock` 就行了，注意，我们每次处理音频的时候都设置了它的 PTS，但是当你看 `audio_callback` 函数的实现时，你会发现它需要花费时间将所有的数据从音频的 packet 移到输出的 buffer 中，这就意味着我们的 audio clock 的值可能会太领先，所以我们要检查我们差了多少时间。这里是代码：


```
double get_audio_clock(VideoState *is) {
	double pts;
	int hw_buf_size, bytes_per_sec, n;
	
	pts = is->audio_clock; // Maintained in the audio thread.
	hw_buf_size = is->audio_buf_size - is->audio_buf_index;
	bytes_per_sec = 0;
	n = is->audio_st->codec->channels * 2;
	if (is->audio_st) {
		bytes_per_sec = is->audio_st->codec->sample_rate * n;
	}
	if (bytes_per_sec) {
		pts -= (double) hw_buf_size / bytes_per_sec;
	}
	return pts;
}
```

现在我们应该能理解这里为什么要这样写了。






以上便是我们这节教程的全部代码，你可以从这里获得：[https://github.com/samirchen/TestFFmpeg][6]

## 编译执行

你可以使用下面的命令编译它：

```
$ gcc -o tutorial05 tutorial05.c -lavutil -lavformat -lavcodec -lswscale -lz -lm `sdl-config --cflags --libs`
```

找一个视频文件，你可以这样执行一下试试：

```
$ tutorial05 myvideofile.mp4
```






[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ffmpeg-tutorial-5
[3]: http://dranger.com/ffmpeg/tutorial05.html
