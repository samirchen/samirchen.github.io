---
layout: post
title: AVAudioFoundation(2)：音视频播放
description: 介绍 iOS 上的音视频播放。
category: blog
tag: Audio, Video, Live, iOS, Recorder, AVFoundation, AVAsset, Playback
---

本文主要内容来自 [AVFoundation Programming Guide][3]。



要播放 `AVAsset` 可以使用 `AVPlayer`。在播放期间，可以使用一个 `AVPlayerItem` 实例来管理 asset 的整体的播放状态，使用 `AVPlayerItemTrack` 来管理各个 track 的播放状态。对于视频的渲染，使用 `AVPlayerLayer` 来处理。


## 播放 Asset


`AVPlayer` 是一个控制 asset 播放的控制器，它的功能包括：开始播放、停止播放、seek 等等。你可以使用 `AVPlayer` 来播放单个 asset。如果你想播放一组 asset，你可以使用 `AVQueuePlayer`，`AVQueuePlayer` 是 `AVPlayer` 的子类。

`AVPlayer` 也会提供当前的播放状态，这样我们就可以根据当前的播放状态调整交互。我们需要将 `AVPlayer` 的画面输出到一个特定的 Core Animation Layer 上，通常是一个 `AVPlayerLayer` 或 `AVSynchronizedLayer` 实例。

需要注意的是，你可以从一个 `AVPlayer` 实例创建多个 `AVPlayerLayer` 对象，但是只有最新创建的那个才会渲染画面到屏幕。







[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-av-playback
[3]: https://developer.apple.com/library/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html