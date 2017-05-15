---
layout: post
title: AVAudioFoundation(3)：音视频编辑
description: 介绍在 iOS 上做音视频录制相关开发的一些基本概念。
category: blog
tag: Audio, Video, Live, iOS, Recorder
---


本文主要内容来自 [AVFoundation Programming Guide][7]。


要了解 iOS 上的音视频编辑相关的内容，首先需要了解的就是 `AVFoundation` 这个框架。

下图是 `AVFoundation` 框架大的层级结构：

![image](../../images/ios-av-edit/avfoundation-stack-on-ios.png)

下图是 `AVFoundation` 框架中各个类的关系结构：

![image](../../images/ios-av-edit/media_trunk_graph.svg)

在 `AVFoundation` 框架中，最主要的表示媒体的类就是 `AVAsset`，甚至可以认为 `AVFoundation` 框架的大部分能力都是围绕着 `AVAsset` 展开的。

一个 `AVAsset` 实例表示的是一份或多份音视频数据（audio and video tracks）的集合，它描述的是这个集合作为一个整体对象的一些属性，比如：标题、时长、大小等，而不与具体的数据格式绑定。通常，在实际使用时我们可能会基于某个 URL 创建对应的媒体资源对象（AVURLAsset），或者直接创建 compositions（AVComposition），这些类都是 `AVAsset` 的子类。

一个 `AVAsset` 中的每一份音频或视频数据都称为一个**轨道（track）**。在最简单的情况下，一个媒体文件中可能只有两个轨道，一个音频轨道，一个视频轨道。而复杂的组合中，可能包含多个重叠的音频轨道和视频轨道。此外 `AVAsset` 也可能包含**元数据（metadata）**。

在 `AVFoundation` 中另一个非常重要的概念是，初始化一个 `AVAsset` 或者一个 `AVAssetTrack` 时并不一定意味着它已经可以立即使用，因为这需要一段时间来做计算，而这个计算可能会阻塞当前线程，所以通常你可以选用异步的方式来初始化，并通过回调来得到异步返回。


## 音视频编辑

上面简单了解了下 `AVFoundation` 框架后，我们来看看跟音视频编辑相关的接口。

一个 composition 可以简单的认为是一组轨道（tracks）的集合，这些轨道可以是来自不同媒体资源（asset）。`AVMutableComposition` 提供了接口来插入或者删除轨道，也可以调整这些轨道的顺序。

下面这张图反映了一个新的 composition 是怎么从已有的 asset 中获取对应的 track 并进行拼接形成新的 asset。

![image](../../images/ios-av-edit/avmutablecomposition.png)

在处理音频时，你可以在使用 `AVMutableAudioMix` 类的接口来做一些自定义的操作，如下图所示。现在，你可以做到指定一个最大音量或设置一个音频轨道的音量渐变。

![image](../../images/ios-av-edit/avmutableaudiomix.png)


如下图所示，我们还可以使用 `AVMutableVideoComposition` 来直接处理 composition 中的视频轨道。处理一个单独的 video composition 时，你可以指定它的渲染尺寸、缩放比例、帧率等参数并输出最终的视频文件。通过一些针对 video composition 的指令（AVMutableVideoCompositionInstruction 等），我们可以修改视频的背景颜色、应用 layer instructions。这些 layer instructions（AVMutableVideoCompositionLayerInstruction 等）可以用来对 composition 中的视频轨道实施图形变换、添加图形渐变、透明度变换、增加透明度渐变。此外，你还能通过设置 video composition 的 `animationTool` 属性来应用 Core Animation Framework 框架中的动画效果。


![image](../../images/ios-av-edit/avmutablevideocomposition.png)


如下图所示，你可以使用 `AVAssetExportSession` 相关的接口来合并你的 composition 中的 audio mix 和 video composition。你只需要初始化一个 `AVAssetExportSession` 对象，然后将其 `audioMix` 和 `videoComposition` 属性分别设置为你的 audio mix 和 video composition 即可。

![image](../../images/ios-av-edit/puttingitalltogether.png)


## 创建 Composition

上面简单介绍了集中音视频编辑的场景，现在我们来详细介绍具体的接口。从 `AVMutableComposition` 开始。

当使用 `AVMutableComposition` 创建自己的 composition 时，最典型的，我们可以使用 `AVMutableCompositionTrack` 来向 composition 中添加一个或多个 composition tracks，比如下面这个简单的例子便是向一个 composition 中添加一个音频轨道和一个视频轨道：

```
AVMutableComposition *mutableComposition = [AVMutableComposition composition];
// Create the video composition track.
AVMutableCompositionTrack *mutableCompositionVideoTrack = [mutableComposition addMutableTrackWithMediaType:AVMediaTypeVideo preferredTrackID:kCMPersistentTrackID_Invalid];
// Create the audio composition track.
AVMutableCompositionTrack *mutableCompositionAudioTrack = [mutableComposition addMutableTrackWithMediaType:AVMediaTypeAudio preferredTrackID:kCMPersistentTrackID_Invalid];
```

当为 composition 添加一个新的 track 的时候，需要设置其媒体类型（media type）和 track ID，主要的媒体类型包括：音频、视频、字幕、文本等等。

这里需要注意的是，每个 track 都需要一个唯一的 track ID，比较方便的做法是：设置 track ID 为 kCMPersistentTrackID_Invalid 来为对应的 track 获得一个自动生成的唯一 ID。


## 向 Composition 添加视听数据

要将媒体数据添加到一个 composition track 中需要访问媒体数据所在的 `AVAsset`，可以使用 `AVMutableCompositionTrack` 的接口将具有相同媒体类型的多个 track 添加到同一个 composition track 中。下面的例子便是从两个 `AVAsset` 中各取出一份 video asset track，再添加到一个新的 composition track 中去：

```
// You can retrieve AVAssets from a number of places, like the camera roll for example.
AVAsset *videoAsset = <#AVAsset with at least one video track#>;
AVAsset *anotherVideoAsset = <#another AVAsset with at least one video track#>;
// Get the first video track from each asset.
AVAssetTrack *videoAssetTrack = [[videoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
AVAssetTrack *anotherVideoAssetTrack = [[anotherVideoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
// Add them both to the composition.
[mutableCompositionVideoTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero,videoAssetTrack.timeRange.duration) ofTrack:videoAssetTrack atTime:kCMTimeZero error:nil];
[mutableCompositionVideoTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero,anotherVideoAssetTrack.timeRange.duration) ofTrack:anotherVideoAssetTrack atTime:videoAssetTrack.timeRange.duration error:nil];
```

## 检索兼容的 Composition Tracks

如果可能，每一种媒体类型最好只使用一个 composition track，这样能够优化资源的使用。当你连续播放媒体数据时，应该将相同类型的媒体数据放到同一个 composition track 中，你可以通过类似下面的代码来从 composition 中查找是否有与当前的 asset track 兼容的 composition track，然后拿来使用：

```
AVMutableCompositionTrack *compatibleCompositionTrack = [mutableComposition mutableTrackCompatibleWithTrack:<#the AVAssetTrack you want to insert#>];
if (compatibleCompositionTrack) {
    // Implementation continues.
}
```

需要注意的是，在同一个 composition track 中添加多个视频段时，当视频段之间切换时可能会丢帧，尤其在嵌入式设备上。基于这个问题，应该合理选择一个 composition track 里的视频段数量。


## 设置音量渐变

只使用一个 `AVMutableAudioMix` 对象就能够为 composition 中的每一个 audio track 单独做音频处理。

下面代码展示了如果使用 `AVMutableAudioMix` 给一个 audio track 设置音量渐变给声音增加一个淡出效果。使用 `audioMix` 类方法获取 `AVMutableAudioMix` 实例；然后使用 `AVMutableAudioMixInputParameters` 类的 `audioMixInputParametersWithTrack:` 接口将 `AVMutableAudioMix` 实例与 composition 中的某一个 audio track 关联起来；之后便可以通过 `AVMutableAudioMix` 实例来处理音量了。

```
AVMutableAudioMix *mutableAudioMix = [AVMutableAudioMix audioMix];
// Create the audio mix input parameters object.
AVMutableAudioMixInputParameters *mixParameters = [AVMutableAudioMixInputParameters audioMixInputParametersWithTrack:mutableCompositionAudioTrack];
// Set the volume ramp to slowly fade the audio out over the duration of the composition.
[mixParameters setVolumeRampFromStartVolume:1.f toEndVolume:0.f timeRange:CMTimeRangeMake(kCMTimeZero, mutableComposition.duration)];
// Attach the input parameters to the audio mix.
mutableAudioMix.inputParameters = @[mixParameters];
```


## 自定义视频处理


处理音频是我们使用 `AVMutableAudioMix`，那么处理视频时，我们就使用 `AVMutableVideoComposition`，只需要一个 `AVMutableVideoComposition` 实例就可以为 composition 中所有的 video track 做处理，比如设置渲染尺寸、缩放、播放帧率等等。

下面我们依次来看一些场景。




### 设置视频背景色

所有的 video composition 也必然对应一组 `AVVideoCompositionInstruction` 实例，每个 `AVVideoCompositionInstruction` 中至少包含一条 video composition instruction。我们可以使用 `AVMutableVideoCompositionInstruction` 来创建我们自己的 video composition instruction，通过这些指令，我们可以修改 composition 的背景颜色、后处理、layer instruction 等等。

下面的实例代码展示了如何创建 video composition instruction 并将一个 composition 的整个时长都设置为红色背景色：

```
AVMutableVideoCompositionInstruction *mutableVideoCompositionInstruction = [AVMutableVideoCompositionInstruction videoCompositionInstruction];
mutableVideoCompositionInstruction.timeRange = CMTimeRangeMake(kCMTimeZero, mutableComposition.duration);
mutableVideoCompositionInstruction.backgroundColor = [[UIColor redColor] CGColor];
```

### 设置透明度渐变

我们也可以用 video composition instructions 来应用 video composition layer instructions。`AVMutableVideoCompositionLayerInstruction` 可以用来设置 video track 的图形变换、图形渐变、透明度、透明度渐变等等。一个 video composition instruction 的 `layerInstructions` 属性中所存储的 layer instructions 的顺序决定了 tracks 中的视频帧是如何被放置和组合的。

下面的示例代码展示了如何在从一个视频切换到第二个视频时添加一个透明度渐变的效果：

```
AVAsset *firstVideoAssetTrack = <#AVAssetTrack representing the first video segment played in the composition#>;
AVAsset *secondVideoAssetTrack = <#AVAssetTrack representing the second video segment played in the composition#>;
// Create the first video composition instruction.
AVMutableVideoCompositionInstruction *firstVideoCompositionInstruction = [AVMutableVideoCompositionInstruction videoCompositionInstruction];
// Set its time range to span the duration of the first video track.
firstVideoCompositionInstruction.timeRange = CMTimeRangeMake(kCMTimeZero, firstVideoAssetTrack.timeRange.duration);
// Create the layer instruction and associate it with the composition video track.
AVMutableVideoCompositionLayerInstruction *firstVideoLayerInstruction = [AVMutableVideoCompositionLayerInstruction videoCompositionLayerInstructionWithAssetTrack:mutableCompositionVideoTrack];
// Create the opacity ramp to fade out the first video track over its entire duration.
[firstVideoLayerInstruction setOpacityRampFromStartOpacity:1.f toEndOpacity:0.f timeRange:CMTimeRangeMake(kCMTimeZero, firstVideoAssetTrack.timeRange.duration)];
// Create the second video composition instruction so that the second video track isn't transparent.
AVMutableVideoCompositionInstruction *secondVideoCompositionInstruction = [AVMutableVideoCompositionInstruction videoCompositionInstruction];
// Set its time range to span the duration of the second video track.
secondVideoCompositionInstruction.timeRange = CMTimeRangeMake(firstVideoAssetTrack.timeRange.duration, CMTimeAdd(firstVideoAssetTrack.timeRange.duration, secondVideoAssetTrack.timeRange.duration));
// Create the second layer instruction and associate it with the composition video track.
AVMutableVideoCompositionLayerInstruction *secondVideoLayerInstruction = [AVMutableVideoCompositionLayerInstruction videoCompositionLayerInstructionWithAssetTrack:mutableCompositionVideoTrack];
// Attach the first layer instruction to the first video composition instruction.
firstVideoCompositionInstruction.layerInstructions = @[firstVideoLayerInstruction];
// Attach the second layer instruction to the second video composition instruction.
secondVideoCompositionInstruction.layerInstructions = @[secondVideoLayerInstruction];
// Attach both of the video composition instructions to the video composition.
AVMutableVideoComposition *mutableVideoComposition = [AVMutableVideoComposition videoComposition];
mutableVideoComposition.instructions = @[firstVideoCompositionInstruction, secondVideoCompositionInstruction];
```


### 动画效果

我们还能通过设置 video composition 的 `animationTool` 属性来使用 Core Animation Framework 框架的强大能力。比如：设置视频水印、视频标题、动画浮层等。

在 video composition 中使用 Core Animation 有两种不同的方式：

- 添加一个 Core Animation Layer 作为独立的 composition track
- 直接使用 Core Animation Layer 在视频帧中渲染动画效果

下面的代码展示了后面一种使用方式，在视频区域的中心添加水印：

```
CALayer *watermarkLayer = <#CALayer representing your desired watermark image#>;
CALayer *parentLayer = [CALayer layer];
CALayer *videoLayer = [CALayer layer];
parentLayer.frame = CGRectMake(0, 0, mutableVideoComposition.renderSize.width, mutableVideoComposition.renderSize.height);
videoLayer.frame = CGRectMake(0, 0, mutableVideoComposition.renderSize.width, mutableVideoComposition.renderSize.height);
[parentLayer addSublayer:videoLayer];
watermarkLayer.position = CGPointMake(mutableVideoComposition.renderSize.width/2, mutableVideoComposition.renderSize.height/4);
[parentLayer addSublayer:watermarkLayer];
mutableVideoComposition.animationTool = [AVVideoCompositionCoreAnimationTool videoCompositionCoreAnimationToolWithPostProcessingAsVideoLayer:videoLayer inLayer:parentLayer];
```



## 一个完整示例


这里的实例将展示如何合并两个 video asset tracks 和一个 audio asset track 到一个视频文件，其中大体步骤如下：

- 创建一个 `AVMutableComposition` 对象，添加多个 `AVMutableCompositionTrack` 对象
- 在各个 composition tracks 中添加 `AVAssetTrack` 对应的时间范围
- 检查 video asset track 的 `preferredTransform` 属性决定视频方向
- 使用 `AVMutableVideoCompositionLayerInstruction` 对象对视频进行图形变换
- 设置 video composition 的 `renderSize` 和 `frameDuration` 属性
- 导出视频文件
- 保存视频文件到相册


下面的示例代码省略了一些内存管理和通知移除相关的代码。


```
// 1、创建 composition。创建一个 composition，并添加一个 audio track 和一个 video track。
AVMutableComposition *mutableComposition = [AVMutableComposition composition];
AVMutableCompositionTrack *videoCompositionTrack = [mutableComposition addMutableTrackWithMediaType:AVMediaTypeVideo preferredTrackID:kCMPersistentTrackID_Invalid];
AVMutableCompositionTrack *audioCompositionTrack = [mutableComposition addMutableTrackWithMediaType:AVMediaTypeAudio preferredTrackID:kCMPersistentTrackID_Invalid];

// 2、添加 asset。从源 assets 中取得两个 video track 和一个 audio track，在上面的 video composition track 中依次添加两个 video track，在 audio composition track 中添加一个 video track。
AVAsset *firstVideoAsset = <#First AVAsset with at least one video track#>;
AVAsset *secondVideoAsset = <#Second AVAsset with at least one video track#>;
AVAsset *audioAsset = <#AVAsset with at least one audio track#>;
AVAssetTrack *firstVideoAssetTrack = [[firstVideoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
AVAssetTrack *secondVideoAssetTrack = [[secondVideoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
AVAssetTrack *audioAssetTrack = [[audioAsset tracksWithMediaType:AVMediaTypeAudio] objectAtIndex:0]
[videoCompositionTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, firstVideoAssetTrack.timeRange.duration) ofTrack:firstVideoAssetTrack atTime:kCMTimeZero error:nil];
[videoCompositionTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, secondVideoAssetTrack.timeRange.duration) ofTrack:secondVideoAssetTrack atTime:firstVideoAssetTrack.timeRange.duration error:nil];
[audioCompositionTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, CMTimeAdd(firstVideoAssetTrack.timeRange.duration, secondVideoAssetTrack.timeRange.duration)) ofTrack:audioAssetTrack atTime:kCMTimeZero error:nil];

// 3、检查 composition 方向。在 composition 中添加了 audio track 和 video track 后，还必须确保其中所有的 video track 的视频方向都是一致的。在默认情况下 video track 默认为横屏模式，如果这时添加进来的 video track 是在竖屏模式下采集的，那么导出的视频会出现方向错误。同理，将一个横向的视频和一个纵向的视频进行合并导出，export session 会报错。
BOOL isFirstVideoPortrait = NO;
CGAffineTransform firstTransform = firstVideoAssetTrack.preferredTransform;
// Check the first video track's preferred transform to determine if it was recorded in portrait mode.
if (firstTransform.a == 0 && firstTransform.d == 0 && (firstTransform.b == 1.0 || firstTransform.b == -1.0) && (firstTransform.c == 1.0 || firstTransform.c == -1.0)) {
    isFirstVideoPortrait = YES;
}
BOOL isSecondVideoPortrait = NO;
CGAffineTransform secondTransform = secondVideoAssetTrack.preferredTransform;
// Check the second video track's preferred transform to determine if it was recorded in portrait mode.
if (secondTransform.a == 0 && secondTransform.d == 0 && (secondTransform.b == 1.0 || secondTransform.b == -1.0) && (secondTransform.c == 1.0 || secondTransform.c == -1.0)) {
    isSecondVideoPortrait = YES;
}
if ((isFirstVideoAssetPortrait && !isSecondVideoAssetPortrait) || (!isFirstVideoAssetPortrait && isSecondVideoAssetPortrait)) {
    UIAlertView *incompatibleVideoOrientationAlert = [[UIAlertView alloc] initWithTitle:@"Error!" message:@"Cannot combine a video shot in portrait mode with a video shot in landscape mode." delegate:self cancelButtonTitle:@"Dismiss" otherButtonTitles:nil];
    [incompatibleVideoOrientationAlert show];
    return;
}

// 4、应用 Video Composition Layer Instructions。一旦你知道你要合并的视频片段的方向是兼容的，那么你接下来就可以为每个片段应用必要的 layer instructions，并将这些 layer instructions 添加到 video composition 中。
// 所有的 `AVAssetTrack` 对象都有一个 `preferredTransform` 属性，包含了 asset track 的方向信息。这个 transform 会在 asset track 在屏幕上展示时被应用。在下面的代码中，layer instruction 的 transform 被设置为 asset track 的 transform，便于在你修改了视频尺寸时，新的 composition 中的视频也能正确的进行展示。
AVMutableVideoCompositionInstruction *firstVideoCompositionInstruction = [AVMutableVideoCompositionInstruction videoCompositionInstruction];
// Set the time range of the first instruction to span the duration of the first video track.
firstVideoCompositionInstruction.timeRange = CMTimeRangeMake(kCMTimeZero, firstVideoAssetTrack.timeRange.duration);
AVMutableVideoCompositionInstruction *secondVideoCompositionInstruction = [AVMutableVideoCompositionInstruction videoCompositionInstruction];
// Set the time range of the second instruction to span the duration of the second video track.
secondVideoCompositionInstruction.timeRange = CMTimeRangeMake(firstVideoAssetTrack.timeRange.duration, CMTimeAdd(firstVideoAssetTrack.timeRange.duration, secondVideoAssetTrack.timeRange.duration));

// 创建两个 video layer instruction，关联对应的 video composition track，并设置 transform 为 preferredTransform。
AVMutableVideoCompositionLayerInstruction *firstVideoLayerInstruction = [AVMutableVideoCompositionLayerInstruction videoCompositionLayerInstructionWithAssetTrack:videoCompositionTrack];
// Set the transform of the first layer instruction to the preferred transform of the first video track.
[firstVideoLayerInstruction setTransform:firstTransform atTime:kCMTimeZero];
AVMutableVideoCompositionLayerInstruction *secondVideoLayerInstruction = [AVMutableVideoCompositionLayerInstruction videoCompositionLayerInstructionWithAssetTrack:videoCompositionTrack];
// Set the transform of the second layer instruction to the preferred transform of the second video track.
[secondVideoLayerInstruction setTransform:secondTransform atTime:firstVideoAssetTrack.timeRange.duration];

firstVideoCompositionInstruction.layerInstructions = @[firstVideoLayerInstruction];
secondVideoCompositionInstruction.layerInstructions = @[secondVideoLayerInstruction];
AVMutableVideoComposition *mutableVideoComposition = [AVMutableVideoComposition videoComposition];
mutableVideoComposition.instructions = @[firstVideoCompositionInstruction, secondVideoCompositionInstruction];

// 5、设置渲染尺寸和帧率。要完全解决视频方向问题，你还需要调整 video composition 的 `renderSize` 属性，同时也需要设置一个合适的 `frameDuration`，比如 1/30 表示 30 帧每秒。此外，`renderScale` 默认值为 1.0。
CGSize naturalSizeFirst, naturalSizeSecond;
// If the first video asset was shot in portrait mode, then so was the second one if we made it here.
if (isFirstVideoAssetPortrait) {
    // Invert the width and height for the video tracks to ensure that they display properly.
    naturalSizeFirst = CGSizeMake(firstVideoAssetTrack.naturalSize.height, firstVideoAssetTrack.naturalSize.width);
    naturalSizeSecond = CGSizeMake(secondVideoAssetTrack.naturalSize.height, secondVideoAssetTrack.naturalSize.width);
} else {
    // If the videos weren't shot in portrait mode, we can just use their natural sizes.
    naturalSizeFirst = firstVideoAssetTrack.naturalSize;
    naturalSizeSecond = secondVideoAssetTrack.naturalSize;
}
float renderWidth, renderHeight;
// Set the renderWidth and renderHeight to the max of the two videos widths and heights.
if (naturalSizeFirst.width > naturalSizeSecond.width) {
    renderWidth = naturalSizeFirst.width;
} else {
    renderWidth = naturalSizeSecond.width;
}
if (naturalSizeFirst.height > naturalSizeSecond.height) {
    renderHeight = naturalSizeFirst.height;
} else {
    renderHeight = naturalSizeSecond.height;
}
mutableVideoComposition.renderSize = CGSizeMake(renderWidth, renderHeight);
// Set the frame duration to an appropriate value (i.e. 30 frames per second for video).
mutableVideoComposition.frameDuration = CMTimeMake(1,30);


// 6、导出 composition 并保持到相册。创建一个 `AVAssetExportSession` 对象，设置对应的 `outputURL` 来将视频导出到指定的文件。同时，我们还可以用 `ALAssetsLibrary` 接口来将导出的视频文件存储到相册中去。
// Create a static date formatter so we only have to initialize it once.
static NSDateFormatter *kDateFormatter;
if (!kDateFormatter) {
    kDateFormatter = [[NSDateFormatter alloc] init];
    kDateFormatter.dateStyle = NSDateFormatterMediumStyle;
    kDateFormatter.timeStyle = NSDateFormatterShortStyle;
}
// Create the export session with the composition and set the preset to the highest quality.
AVAssetExportSession *exporter = [[AVAssetExportSession alloc] initWithAsset:mutableComposition presetName:AVAssetExportPresetHighestQuality];
// Set the desired output URL for the file created by the export process.
exporter.outputURL = [[[[NSFileManager defaultManager] URLForDirectory:NSDocumentDirectory inDomain:NSUserDomainMask appropriateForURL:nil create:@YES error:nil] URLByAppendingPathComponent:[kDateFormatter stringFromDate:[NSDate date]]] URLByAppendingPathExtension:CFBridgingRelease(UTTypeCopyPreferredTagWithClass((CFStringRef)AVFileTypeQuickTimeMovie, kUTTagClassFilenameExtension))];
// Set the output file type to be a QuickTime movie.
exporter.outputFileType = AVFileTypeQuickTimeMovie;
exporter.shouldOptimizeForNetworkUse = YES;
exporter.videoComposition = mutableVideoComposition;
// Asynchronously export the composition to a video file and save this file to the camera roll once export completes.
[exporter exportAsynchronouslyWithCompletionHandler:^{
    dispatch_async(dispatch_get_main_queue(), ^{
        if (exporter.status == AVAssetExportSessionStatusCompleted) {
            ALAssetsLibrary *assetsLibrary = [[ALAssetsLibrary alloc] init];
            if ([assetsLibrary videoAtPathIsCompatibleWithSavedPhotosAlbum:exporter.outputURL]) {
                [assetsLibrary writeVideoAtPathToSavedPhotosAlbum:exporter.outputURL completionBlock:NULL];
            }
        }
    });
}];
```


## 其他参考

- [一个简单的视频编辑 Demo][4]



[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-av-edit
[3]: http://www.jianshu.com/p/5433143cccd8
[4]: https://developer.apple.com/library/content/samplecode/AVSimpleEditoriOS/Introduction/Intro.html
[5]: http://www.jianshu.com/p/02e872ecf0d1
[6]: http://yoferzhang.com/post/20160724AVFoundation/
[7]: https://developer.apple.com/library/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html

<!-- AVCam-iOS: Using AVFoundation to Capture Images and Movies is the canonical sample code for implementing any program that uses the camera functionality. It is a complete sample, well documented, and covers the majority of the functionality showing the best practices. -->
[8]: https://developer.apple.com/library/content/samplecode/AVCam/Introduction/Intro.html

<!-- AVCamManual: Extending AVCam to Use Manual Capture API is the companion application to AVCam. It implements Camera functionality using the manual camera controls. It is also a complete example, well documented, and should be considered the canonical example for creating camera applications that take advantage of manual controls. -->
[9]: https://developer.apple.com/library/content/samplecode/AVCamManual/Introduction/Intro.html

<!-- RosyWriter is an example that demonstrates real time frame processing and in particular how to apply filters to video content. This is a very common developer requirement and this example covers that functionality. -->
[10]: https://developer.apple.com/library/content/samplecode/RosyWriter/Introduction/Intro.html

<!-- AVLocationPlayer: Using AVFoundation Metadata Reading APIs demonstrates using the metadata APIs. -->
[11]: https://developer.apple.com/library/content/samplecode/AVLocationPlayer/Introduction/Intro.html
