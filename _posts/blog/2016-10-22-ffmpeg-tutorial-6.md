---
layout: post
title: FFmpeg 入门(6)：音频同步
description: FFmpeg 入门教程系列。
category: blog
tag: FFmpeg, tutorial
---


## 音频同步

上一节我们做了将视频同步到音频时钟，这一节我们反过来，将音频同步到视频。首先，我们要实现一个视频时钟来跟踪视频线程播放了多久，并将音频同步过来。后面我们会看看如何将音频和视频都同步到外部时钟。


## 实现视频时钟


与音频时钟类似，我们现在要实现一个视频时钟：即一个内部的值来记录视频已经播放的时间。首先，你可能会认为就是简单地根据被显示的最后一帧的 PTS 值来更新一下时间就可以了。但是，不要忘了当我们以毫秒作为衡量单位时视频帧之间的间隔可能会很大的。所以解决方案是跟踪另一个值：我们将视频时钟设置为最后一帧的 PTS 时的时间。这样当前的视频时钟的值就应该是 `PTS_of_last_frame + (current_time - time_elapsed_since_PTS_value_was_set)`。这个方案和我们前面实现的 `get_audio_clock()` 类似。



所以，在 `VideoState` 结构体中我们要添加成员 `double video_current_pts` 和 `int64_t video_current_pts_time`，时钟的更新会在 `video_refresh_timer()` 函数中进行：


```
void video_refresh_timer(void *userdata) {

	// ... code ...
	
	if (is->video_st) {
		if (is->pictq_size == 0) {
			schedule_refresh(is, 1);
		} else {
			vp = &is->pictq[is->pictq_rindex];
			
			is->video_current_pts = vp->pts;
			is->video_current_pts_time = av_gettime();


	// ... code ...

}
```			


不要忘了在 `stream_component_open()` 中初始化它：

```
is->video_current_pts_time = av_gettime();
```

我们接着就实现 `get_video_clock()`：

```
double get_video_clock(VideoState *is) {
	double delta;
	
	delta = (av_gettime() - is->video_current_pts_time) / 1000000.0;
	return is->video_current_pts + delta;
}
```


## 抽象和封装时钟获取函数


有一点需要我们考虑的是我们不应该把代码写的太耦合，否则当我们需要修改音视频同步逻辑为同步外部时钟时，我们就得修改代码。那在像 FFPlay 那样可以通过命令行选项控制的场景下，就乱套了。所以这里我们要做一些抽象和封装的工作：实现一个包装函数 `get_master_clock()` 通过检查 `av_sync_type` 选项的值来决定该选择哪一个时钟作为同步的基准，从而决定去调用 `get_audio_clock` 或 `get_video_clock` 还是其他 clock。我们甚至可以使用系统时钟，这里我们叫做 `get_external_clock`。


```
enum {
	AV_SYNC_AUDIO_MASTER,
	AV_SYNC_VIDEO_MASTER,
	AV_SYNC_EXTERNAL_MASTER,
};

#define DEFAULT_AV_SYNC_TYPE AV_SYNC_VIDEO_MASTER

double get_master_clock(VideoState *is) {
	if (is->av_sync_type == AV_SYNC_VIDEO_MASTER) {
		return get_video_clock(is);
	} else if (is->av_sync_type == AV_SYNC_AUDIO_MASTER) {
		return get_audio_clock(is);
	} else {
		return get_external_clock(is);
	}
}


int main(int argc, char *argv[]) {
	// ... code ...

	is->av_sync_type = DEFAULT_AV_SYNC_TYPE;

	// ... code ...
}
```


## 音频同步实现


现在来到了最难的部分：同步音频到视频时钟。我们的策略是计算音频播放的时间点，然后跟视频时钟做比较，然后计算我们要调整多少个音频采样，也就是：我们需要丢掉多少采样来加速让音频追赶上视频时钟或者我们要添加多少采样来降速来等待视频时钟。

我们要实现一个 `synchronize_audio()` 函数，在每次处理一组音频采样时去调用它来丢弃音频采样或者拉伸音频采样。但是，我们也不希望一不同步就处理，因为毕竟音频处理的频率比视频要多很多，所以我们会设置一个值来约束连续调用 `synchronize_audio()` 的次数。当然和前面一样，这里的不同步是指音频时钟和视频时钟的差值超过了我们的阈值。


现在让我们看看当 N 组音频采样已经不同步的情况。而这些音频采样不同步的程度也有很大的不同，所以我们要取平均值来衡量每个采样的不同步情况。比如，第一次调用时显示我们不同步了 40ms，下一次是 50ms，等等。但是我们不会采取简单的平均计算，因为最近的值比之前的值更重要也更有意义，这时候我们会使用一个小数系数 `c`，并对不同步的延时求和：`diff_sum = new_diff + diff_sum * c`。当我们找到平均差异值时，我们就简单的计算 `avg_diff = diff_sum * (1 - c)`。我们代码如下：


```
// Add or subtract samples to get a better sync, return new audio buffer size.
int synchronize_audio(VideoState *is, short *samples, int samples_size, double pts) {
	int n;
	double ref_clock;
	
	n = 2 * is->audio_st->codec->channels;
	
	if (is->av_sync_type != AV_SYNC_AUDIO_MASTER) {
		double diff, avg_diff;
		int wanted_size, min_size, max_size; //, nb_samples 
		
		ref_clock = get_master_clock(is);
		diff = get_audio_clock(is) - ref_clock;
		
		if (diff < AV_NOSYNC_THRESHOLD) {
			// Accumulate the diffs.
			is->audio_diff_cum = diff + is->audio_diff_avg_coef
			* is->audio_diff_cum;
			if (is->audio_diff_avg_count < AUDIO_DIFF_AVG_NB) {
				is->audio_diff_avg_count++;
			} else {
				avg_diff = is->audio_diff_cum * (1.0 - is->audio_diff_avg_coef);
				if (fabs(avg_diff) >= is->audio_diff_threshold) {
					wanted_size = samples_size + ((int) (diff * is->audio_st->codec->sample_rate) * n);
					min_size = samples_size * ((100 - SAMPLE_CORRECTION_PERCENT_MAX) / 100);
					max_size = samples_size * ((100 + SAMPLE_CORRECTION_PERCENT_MAX) / 100);
					if (wanted_size < min_size) {
						wanted_size = min_size;
					} else if (wanted_size > max_size) {
						wanted_size = max_size;
					}
					if (wanted_size < samples_size) {
						// Remove samples.
						samples_size = wanted_size;
					} else if (wanted_size > samples_size) {
						uint8_t *samples_end, *q;
						int nb;
						
						// Add samples by copying final sample.
						nb = (samples_size - wanted_size);
						samples_end = (uint8_t *)samples + samples_size - n;
						q = samples_end + n;
						while (nb > 0) {
							memcpy(q, samples_end, n);
							q += n;
							nb -= n;
						}
						samples_size = wanted_size;
					}
				}
			}
		} else {
			// Difference is too big, reset diff stuff.
			is->audio_diff_avg_count = 0;
			is->audio_diff_cum = 0;
		}
	}
	return samples_size;
}
```

这样一来，我们就知道音频和视频不同步时间的近似值了，我们也知道我们的时钟使用的是什么值来计算。所以接下来我们要计算要丢弃或增加多少个音频采样。「Shrinking/expanding buffer code」 部分即：


```
if (fabs(avg_diff) >= is->audio_diff_threshold) {
	wanted_size = samples_size + ((int) (diff * is->audio_st->codec->sample_rate) * n);
	min_size = samples_size * ((100 - SAMPLE_CORRECTION_PERCENT_MAX) / 100);
	max_size = samples_size * ((100 + SAMPLE_CORRECTION_PERCENT_MAX) / 100);
	if (wanted_size < min_size) {
		wanted_size = min_size;
	} else if (wanted_size > max_size) {
		wanted_size = max_size;
	}

	// ... code ...
```					

注意 `audio_length * (sample_rate * # of channels * 2)` 是时长为 `audio_length` 的音频中采样的数量。因此，我们想要的采样数将是已有的采样数量加上或减去对应于音频偏移的时长的采样数量。我们还会对我们的修正值做一个上限和下限，否则当我们的修正值太大，对用户来说就太刺激了。


## 修正音频采样数

现在我们要着手校正音频了。你可能已经注意到，我们的 `synchronize_audio` 函数返回一个采样的大小，这个是告诉我们要发送到流的字节数。因此，我们只需要将采样大小调整为 `wanted_size`，这样就可以减少采样数。但是，如果我们想要增大采样数，我们不能只是使这个 size 变大，因为这时并没有更多的对应数据在缓冲区！所以我们必须添加采样。但我们应该添加什么采样呢？尝试推算音频是不靠谱的，所以使用已经有的音频来填充即可。这里我们用最后一个音频采样的值填充缓冲区。


```
if (wanted_size < samples_size) {
	// Remove samples.
	samples_size = wanted_size;
} else if (wanted_size > samples_size) {
	uint8_t *samples_end, *q;
	int nb;
	
	// Add samples by copying final sample.
	nb = (samples_size - wanted_size);
	samples_end = (uint8_t *) samples + samples_size - n;
	q = samples_end + n;
	while (nb > 0) {
		memcpy(q, samples_end, n);
		q += n;
		nb -= n;
	}
	samples_size = wanted_size;
}
```

在上面的函数里我们返回了采样的尺寸，现在我们要做的就是用好它：



```
void audio_callback(void *userdata, Uint8 *stream, int len) {
	VideoState *is = (VideoState *)userdata;
	int len1, audio_size;
	double pts;
	
	while (len > 0) {
		if (is->audio_buf_index >= is->audio_buf_size) {
			// We have already sent all our data; get more.
			audio_size = audio_decode_frame(is, &pts);
			if (audio_size < 0) {
				// If error, output silence.
				is->audio_buf_size = 1024;
				memset(is->audio_buf, 0, is->audio_buf_size);
			} else {
				audio_size = synchronize_audio(is, (int16_t *)is->audio_buf, audio_size, pts);
				is->audio_buf_size = audio_size;

	// ... code ...
```				

我们在这里做的就是插入对 `synchronize_audio()` 的调用，当然也要检查一下这里用到的变量的初始化相关的代码。


最后，我们需要确保当视频时钟作为参考时钟时，我们不去做视频同步操作：


```
// Update delay to sync to audio if not master source.
if (is->av_sync_type != AV_SYNC_VIDEO_MASTER) {
	ref_clock = get_master_clock(is);
	diff = vp->pts - ref_clock;
	
	// Skip or repeat the frame. Take delay into account FFPlay still doesn't "know if this is the best guess.".
	sync_threshold = (delay > AV_SYNC_THRESHOLD) ? delay : AV_SYNC_THRESHOLD;
	if (fabs(diff) < AV_NOSYNC_THRESHOLD) {
		if (diff <= -sync_threshold) {
			delay = 0;
		} else if (diff >= sync_threshold) {
			delay = 2 * delay;
		}
	}
}
```








以上便是我们这节教程的全部内容，其中的完整代码你可以从这里获得：[https://github.com/samirchen/TestFFmpeg][6]

## 编译执行

你可以使用下面的命令编译它：

```
$ gcc -o tutorial06 tutorial06.c -lavutil -lavformat -lavcodec -lswscale -lz -lm `sdl-config --cflags --libs`
```

找一个视频文件，你可以这样执行一下试试：

```
$ tutorial06 myvideofile.mp4
```




[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ffmpeg-tutorial-6
[3]: http://dranger.com/ffmpeg/tutorial06.html
[6]: https://github.com/samirchen/TestFFmpeg