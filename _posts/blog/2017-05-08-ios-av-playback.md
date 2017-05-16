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






[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-av-playback
[3]: https://developer.apple.com/library/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html