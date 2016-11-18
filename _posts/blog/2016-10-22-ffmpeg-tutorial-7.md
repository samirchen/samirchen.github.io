---
layout: post
title: FFmpeg 入门(7)：Seeking
description: FFmpeg 入门教程系列。
category: blog
tag: FFmpeg, tutorial
---

## 处理 seek 命令

我们将为播放器添加 seek 的能力。这个过程中，我们会看到 `av_seek_frame` 用起来有多方便。

我们添加的功能是通过上下左右键能够做快进或快退，其中左右键快进或快退的幅度较小，为 10s，上下键快进或快退的幅度较大，为 60s。所以我们需要在我们的事件处理循环中添加处理按键的逻辑。但是当我们遇到按键事件时，我们不能直接调用 `av_seek_frame`，我们需要在 decode loop 和 decode_thread loop 进行处理。所以我们需呀在 `VideoState` 里添加一些变量来记录 seek 的位置以及一些 seek 的标记。


```
typedef struct VideoState {
	// ... code ...

	int seek_req;
	int seek_flags;
	int64_t seek_pos;

	// ... code ...
}
```

我们需要在主函数的事件循环中监听按键事件：

```
for (;;) {
	double incr, pos;
	SDL_WaitEvent(&event);
	switch (event.type) {
		case SDL_KEYDOWN:
			switch (event.key.keysym.sym) {
				case SDLK_LEFT:
					incr = -10.0;
					goto do_seek;
				case SDLK_RIGHT:
					incr = 10.0;
					goto do_seek;
				case SDLK_UP:
					incr = 60.0;
					goto do_seek;
				case SDLK_DOWN:
					incr = -60.0;
					goto do_seek;
				do_seek:
					if (global_video_state) {
						pos = get_master_clock(global_video_state);
						pos += incr;
						stream_seek(global_video_state, (int64_t)(pos * AV_TIME_BASE), incr);
					}
					break;
				default:
					break;
			}
			break;

	// ... code ...
}
```				

当我们监听到按键事件并判断出按键方向时，我们就知道了我们该如何 seek 了，这时候我们通过 `get_master_clock()` 获得此时的时钟值，并加上要 seek 的时间，然后调用 `stream_seek()` 函数来设置 `seek_pos` 等值。我们转换新的时间为 `avcodec` 的内部时间戳单位。记住，在流中时间戳是通过帧数来度量而不是秒，公式是：`seconds = frames * time_base (fps)`。`avcodec` 中 `time_base` 的默认值是 1000000 fps（也就是说 2s 的位置即时间戳为 2000000）。我们将看到我们为什么要转换这个值。


下面是 `stream_seek()` 函数，我们设置了一个 flag 来标记是快进还是快退：

```
void stream_seek(VideoState *is, int64_t pos, int rel) {
	if (!is->seek_req) {
		is->seek_pos = pos;
		is->seek_flags = rel < 0 ? AVSEEK_FLAG_BACKWARD : 0;
		is->seek_req = 1;
	}
}
```


现在我们回到 `decode_thread()`，我们将在这里做实际的 seek 操作。

我们的 seek 操作是围绕着 `av_seek_frame()` 函数进行的。这个函数的参数是：`AVFormatContext *s, int stream_index, int64_t timestamp, int flags`。这个函数将 seek 到你给它的 `timestamp`。`timestamp` 的单位是你传入的流的 `time_base`。但是，你可以不用传入一个流，通过传一个 -1 来表示。如果这样的话，`time_base` 就会是 `avcodec` 内部的时间戳单位，即 1000000 fps。这就是为什么我们要在设置 `seek_pos` 时把 position 乘上 `AV_TIME_BASE` 的原因。


然而，有时候对于有些媒体文件，你传给 `av_seek_frame()` -1 作为 `stream_index` 可能会遇到一些问题。所以我们将选择文件中的第一个流传给 `av_seek_frame()`。不要忘记，这时我们也必须调整我们的时间戳到新的单位。



```
// Seek stuff goes here.
if (is->seek_req) {
	int stream_index= -1;
	int64_t seek_target = is->seek_pos;
	
	if (is->videoStream >= 0) {
		stream_index = is->videoStream;
	} else if (is->audioStream >= 0) {
		stream_index = is->audioStream;
	}
	
	if (stream_index >= 0){
		seek_target= av_rescale_q(seek_target, AV_TIME_BASE_Q, pFormatCtx->streams[stream_index]->time_base);
	}
	if (av_seek_frame(is->pFormatCtx, stream_index, seek_target, is->seek_flags) < 0) {
		fprintf(stderr, "%s: error while seeking\n", is->pFormatCtx->filename);
	} else {

	// ... code ...

}
```				

`av_rescale_q(a, b, c)` 这个函数可以将一个时间戳从一个基址调整到另一个基址。它只是简单的计算 `a * b / c`，但是这个函数是必须的，因为该计算可能溢出。`AV_TIME_BASE_Q` 是 `AV_TIME_BASE` 的分数版本，他们完全不同：`AV_TIME_BASE * time_in_seconds = avcodec_timestamp` 以及 `AV_TIME_BASE_Q * avcodec_timestamp = time_in_seconds`。但要注意，`AV_TIME_BASE_Q` 实际上是一个 `AVRational` 对象，因此你必须在 `avcodec` 中使用特殊的 q 函数来处理它。



## 刷新缓冲区

我们将 seek 调整完毕，但是还没完全完成。我们有一个存放 packet 的队列，既然现在我们 seek 到别的位置了，那么我们就需要刷新一下这个队列，否则视频就没法 seek 了。不光如此，`avcodec` 也有它内部的缓冲区，也需要由各对应的线程来刷新。

为此，首先，我们需要写一个函数来清理我们的 packet 队列。然后，我们需要有一些机制来告诉音视频线程来刷新 `avcodec` 的内部缓冲区。我们可以通过在刷新后的 packet 队列中放一个特殊的 packet 来做到这一点，当 `avcodec` 探测到这个 packet 时，它就会刷新自己的缓冲区。

我们实现的函数是 `packet_queue_flush()`，代码如下：

```
static void packet_queue_flush(PacketQueue *q) {
	AVPacketList *pkt, *pkt1;
	
	SDL_LockMutex(q->mutex);
	for (pkt = q->first_pkt; pkt != NULL; pkt = pkt1) {
		pkt1 = pkt->next;
		av_packet_unref(&pkt->pkt);
		av_freep(&pkt);
	}
	q->last_pkt = NULL;
	q->first_pkt = NULL;
	q->nb_packets = 0;
	q->size = 0;
	SDL_UnlockMutex(q->mutex);
}
```

既然现在队列已经刷新，接着就是放一个 flush packet，但是首先我们要定义一下它：

```
AVPacket flush_pkt;

int main(int argc, char *argv[]) {

	// ... code ...

	av_init_packet(&flush_pkt);
	flush_pkt.data = (unsigned char *) "FLUSH";

	// ... code ...

}
```

现在我们把它放到刷新后的队列中：

```
// Seek stuff goes here.
if (is->seek_req) {

	// ... code ...

	} else {
		if (is->audioStream >= 0) {
			packet_queue_flush(&is->audioq);
			packet_queue_put(&is->audioq, &flush_pkt);
		}
		if (is->videoStream >= 0) {
			packet_queue_flush(&is->videoq);
			packet_queue_put(&is->videoq, &flush_pkt);
		}
	}
	is->seek_req = 0;
}
```


上面这段代码在 `decode_thread()` 中。我们还需要修改一下 `packet_queue_put()` 函数当是 flush packet 时不做拷贝：


```
int packet_queue_put(PacketQueue *q, AVPacket *pkt) {
	AVPacketList *pkt1;
	if (pkt != &flush_pkt && av_packet_ref(pkt, pkt) < 0) {
		return -1;
	}

	// ... code ...

}
```

接着是修改 audio thread 和 video thread，我们在调用 `packet_queue_put()` 后即检查 flush packet 并调用 `avcodec_flush_buffers()`：

视频线程，在 `video_thread()` 中：

```
if (packet_queue_get(&is->videoq, packet, 1) < 0) {
	// Means we quit getting packets.
	break;
}
if (packet->data == flush_pkt.data) {
	avcodec_flush_buffers(is->video_st->codec);
	continue;
}
```

音频线程，在 `audio_decode_frame()` 中：

```
// Next packet.
if (packet_queue_get(&is->audioq, pkt, 1) < 0) {
	return -1;
}
if (pkt->data == flush_pkt.data) {
	avcodec_flush_buffers(is->audio_st->codec);
	continue;
}
```		






以上便是我们这节教程的全部内容，其中的完整代码你可以从这里获得：[https://github.com/samirchen/TestFFmpeg][6]

## 编译执行

你可以使用下面的命令编译它：

```
$ gcc -o tutorial07 tutorial07.c -lavutil -lavformat -lavcodec -lswscale -lz -lm `sdl-config --cflags --libs`
```

找一个视频文件，你可以这样执行一下试试：

```
$ tutorial07 myvideofile.mp4
```





[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ffmpeg-tutorial-7
[3]: http://dranger.com/ffmpeg/tutorial07.html
[6]: https://github.com/samirchen/TestFFmpeg