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


对于 `AVPlayer` 来说，虽然最终播放的是 asset，但是我们并不直接提供一个 `AVAsset` 给它，而是提供一个 `AVPlayerItem` 实例。`AVPlayerItem` 是用来管理与之关联的 asset 的播放状态的，一个 `AVPlayerItem` 包含了一组 `AVPlayerItemTrack` 实例，对应着 asset 中的音视频轨道。它们直接的关系大致如下图所示：

![image](../../images/ios-avfoundation/avplayer_layer.png)

注意：该图的原图是苹果官方文档上的，但是原图是有错的，把 `AVPlayerItemTrack` 所属的框标成了 `AVAsset`，这里做了修正。



这种实现方式就意味着，我们可以用多个播放器同时播放一个 asset，并且各个播放器可以使用不同的模式来渲染。下图就展示了一种用两个不同的 `AVPlayer` 采用不同的设置播放同一个 `AVAsset` 的场景。在播放中，还可以禁掉某些 track 的播放。

![image](../../images/ios-avfoundation/player_objects.png)


我们可以通过网络来加载 asset，通常简单的初始化 `AVPlayerItem` 后并不意味着它就直接能播放，所以我们可以 KVO `AVPlayerItem` 的 `status` 属性来监听它是否已经可播再决定后续的行为。


## 处理不同类型的 Asset








[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-av-playback
[3]: https://developer.apple.com/library/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html