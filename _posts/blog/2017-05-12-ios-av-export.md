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


### 创建 Asset Reader

初始化 `AVAssetReader` 时需要传入相应读取的 asset。


```
NSError *outError;
AVAsset *someAsset = <#AVAsset that you want to read#>;
AVAssetReader *assetReader = [AVAssetReader assetReaderWithAsset:someAsset error:&outError];
BOOL success = (assetReader != nil);
```

需要注意的是要检查创建的 asset reader 是否为空。初始化失败时，可以查看具体的 error 信息。




### 创建 Asset Reader Outputs

在完成创建 asset reader 后，创建至少一个 output 对象来接收读取的媒体数据。当创建 output 对象时，要记得设置 `alwaysCopiesSampleData` 属性为 `NO`，这样你会获得性能上的提升。在本章中的示例代码中，这个属性都应该被设置为 `NO`。


如果我们想从媒体资源中读取一个或多个轨道，并且可能会转换数据的格式，那么可以使用 `AVAssetReaderTrackOutput` 类，为每个 `AVAssetTrack` 轨道使用一个 track output 对象。


下面的示例展示了使用 asset reader 来把一个 audio track 压缩为线性的 PCM：


```
AVAsset *localAsset = assetReader.asset;
// Get the audio track to read.
AVAssetTrack *audioTrack = [[localAsset tracksWithMediaType:AVMediaTypeAudio] objectAtIndex:0];
// Decompression settings for Linear PCM.
NSDictionary *decompressionAudioSettings = @{AVFormatIDKey: [NSNumber numberWithUnsignedInt:kAudioFormatLinearPCM]};
// Create the output with the audio track and decompression settings.
AVAssetReaderOutput *trackOutput = [AVAssetReaderTrackOutput assetReaderTrackOutputWithTrack:audioTrack outputSettings:decompressionAudioSettings];
// Add the output to the reader if possible.
if ([assetReader canAddOutput:trackOutput]) {
    [assetReader addOutput:trackOutput];
}
```


如果想从一个指定的 asset track 按它原本存储的格式读取媒体数据，设置 `outputSettings` 为 `nil` 即可。




当想要读取用 `AVAudioMix` 或 `AVVideoComposition` 混音或编辑过的媒体数据时，需要对应的使用 `AVAssetReaderAudioMixOutput` 和 `AVAssetReaderVideoCompositionOutput`。一般，当我们从一个 `AVComposition` 读取媒体资源时，我们需要用到这些 output 类。





使用一个单独的 audio mix output 我们就可以读取 `AVAudioMix` 混合过的 asset 中的多个音频轨道。为了指明这些音频轨道是如何混合的，我们需要在 `AVAssetReaderAudioMixOutput` 对象初始化完成后将对应的 `AVAudioMix` 对象设置给它的 `audioMix` 属性。

下面的代码展示了如何基于一个 asset 来创建我们的 audio mix output 对象去处理所有的音频轨道，然后压缩这些音频轨道为线性 PCM 数据，并为 output 对象设置 `audioMix` 属性。


```
AVAudioMix *audioMix = <#An AVAudioMix that specifies how the audio tracks from the AVAsset are mixed#>;
// Assumes that assetReader was initialized with an AVComposition object.
AVComposition *composition = (AVComposition *) assetReader.asset;
// Get the audio tracks to read.
NSArray *audioTracks = [composition tracksWithMediaType:AVMediaTypeAudio];
// Get the decompression settings for Linear PCM.
NSDictionary *decompressionAudioSettings = @{AVFormatIDKey: [NSNumber numberWithUnsignedInt:kAudioFormatLinearPCM]};
// Create the audio mix output with the audio tracks and decompression setttings.
AVAssetReaderOutput *audioMixOutput = [AVAssetReaderAudioMixOutput assetReaderAudioMixOutputWithAudioTracks:audioTracks audioSettings:decompressionAudioSettings];
// Associate the audio mix used to mix the audio tracks being read with the output.
audioMixOutput.audioMix = audioMix;
// Add the output to the reader if possible.
if ([assetReader canAddOutput:audioMixOutput]) {
    [assetReader addOutput:audioMixOutput];
}
```


如果想要 asset reader 返回无压缩格式，那么就把 `audioSettings` 参数传为 `nil`。对于使用 `AVAssetReaderVideoCompositionOutput` 时，也是同样的道理。



我们使用 video composition output 的方式跟上面类似。下面的代码展示了，如果读取多个视频轨道的媒体数据并将他们压缩为 ARGB 格式。


```
AVVideoComposition *videoComposition = <#An AVVideoComposition that specifies how the video tracks from the AVAsset are composited#>;
// Assumes assetReader was initialized with an AVComposition.
AVComposition *composition = (AVComposition *) assetReader.asset;
// Get the video tracks to read.
NSArray *videoTracks = [composition tracksWithMediaType:AVMediaTypeVideo];
// Decompression settings for ARGB.
NSDictionary *decompressionVideoSettings = @{(id) kCVPixelBufferPixelFormatTypeKey: [NSNumber numberWithUnsignedInt:kCVPixelFormatType_32ARGB], (id) kCVPixelBufferIOSurfacePropertiesKey: [NSDictionary dictionary]};
// Create the video composition output with the video tracks and decompression setttings.
AVAssetReaderOutput *videoCompositionOutput = [AVAssetReaderVideoCompositionOutput assetReaderVideoCompositionOutputWithVideoTracks:videoTracks videoSettings:decompressionVideoSettings];
// Associate the video composition used to composite the video tracks being read with the output.
videoCompositionOutput.videoComposition = videoComposition;
// Add the output to the reader if possible.
if ([assetReader canAddOutput:videoCompositionOutput]) {
    [assetReader addOutput:videoCompositionOutput];
}
```


### 读取 Asset 的媒体数据


To start reading after setting up all of the outputs you need, call the startReading method on your asset reader. Next, retrieve the media data individually from each output using the copyNextSampleBuffer method. To start up an asset reader with a single output and read all of its media samples, do the following:


在 output 对象创建完成后，接着就要开始读取数据了，这时候我们需要调用 asset reader 的 `startReading` 接口。接着，使用 `copyNextSampleBuffer` 接口来从各个 output 来获取媒体数据。

下面的代码展示了 asset reader 如何用一个 output 对象从 asset 中读取所有的媒体数据：



```
// Start the asset reader up.
[self.assetReader startReading];
BOOL done = NO;
while (!done) {
    // Copy the next sample buffer from the reader output.
    CMSampleBufferRef sampleBuffer = [self.assetReaderOutput copyNextSampleBuffer];
    if (sampleBuffer) {
        // Do something with sampleBuffer here.
        CFRelease(sampleBuffer);
        sampleBuffer = NULL;
    } else {
        // Find out why the asset reader output couldn't copy another sample buffer.
        if (self.assetReader.status == AVAssetReaderStatusFailed) {
            NSError *failureError = self.assetReader.error;
            // Handle the error here.
        } else {
            // The asset reader output has read all of its samples.
            done = YES;
        } 
    }
}
```


## 写入 Asset


`AVAssetWriter` 类可以将媒体数据从多个源写入指定文件格式的单个文件。我们也不需要将 asset writer 对象与特定 asset 相关联，但必须为要创建的每个输出文件使用单独的 asset writer。由于 asset writer 可以从多个来源写出媒体数据，因此我们必须为每个要被写入到输出文件的轨道创建一个 `AVAssetWriterInput` 对象。每个 `AVAssetWriterInput` 对象都期望以 `CMSampleBufferRef` 对象的形式接收数据，但是如果要将 `CVPixelBufferRef` 对象附加到 asset writer，可以使用 `AVAssetWriterInputPixelBufferAdaptor` 类。










[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-av-asset
[3]: https://developer.apple.com/library/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html