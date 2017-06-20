---
layout: post
title: AVAudioFoundation(5)：音视频导出
description: 介绍 iOS 上的音视频导出。
category: blog
tag: Audio, Video, Live, iOS, Recorder, AVFoundation, AVAsset
---

本文主要内容来自 [AVFoundation Programming Guide][3]。


要读写音视频数据资源 asset，我们需要用到 AVFoundation 提供的文件导出 API。`AVAssetExportSession` 提供了比较简单的 API 来满足基本的导出需求，比如修改文件类型、剪辑资源长度。如果要满足更加深度的导出需求，我们则需要用到 `AVAssetReader` 和 `AVAssetWriter`。


当我们需要去操作 asset 的内容时，我们可以用 `AVAssetReader`，比如读取 asset 中的音频轨道来展示波形等等。当我们想用一些音频采样或静态图片去生成 asset 时，我们可以使用 `AVAssetWriter`。

需要注意的是 `AVAssetReader` 不适用于做实时处理。`AVAssetReader` 没法用来处理 HLS 之类的实时数据流。但是 `AVAssetWriter` 是可以用来处理实时数据源的，比如 `AVCaptureOutput`，当需要处理实时数据源时，需要设置 `expectsMediaDataInRealTime` 属性为 `YES`。如果对非实时数据源设置该属性为 `YES`，那么可能会造成你导出的文件有问题。



## 读取 Asset

每一个 `AVAssetReader` 一次只能与一个 asset 关联，但是这个 asset 可以包含多个轨道。由于这个原因通常我们需要为 `AVAssetReader` 指定一个 `AVAssetReaderOutput` 的具体子类来具体操作 asset 的读取，比如：


- `AVAssetReaderTrackOutput`
- `AVAssetReaderAudioMixOutput`
- `AVAssetReaderVideoCompositionOutput`







[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-av-asset
[3]: https://developer.apple.com/library/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html