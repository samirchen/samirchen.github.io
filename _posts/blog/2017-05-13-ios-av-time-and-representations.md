---
layout: post
title: AVAudioFoundation(6)：时间和媒体表示
description: 介绍 iOS 上的音视频时间和媒体表示。
category: blog
tag: Audio, Video, Live, iOS, Recorder, AVFoundation, AVAsset
---

本文主要内容来自 [AVFoundation Programming Guide][3]。


基于时间的音视频数据，例如电影文件或视频流，在 AVFoundation 框架中用 `AVAsset` 来表示。AV Foundation 用于表示时间和媒体的几个底层数据结构，来自 Core Media 框架。


## 资源的表示方式

`AVAsset` 是 AVFoundation 框架中的核心类。它对基于时间的音视频数据的进行了抽象，例如电影文件或视频流。主要关系如下图所示。在许多情况下，我们需要使用其子类之一。


![image](../../images/ios-avfoundation/avasset_hierarchy.png)


Asset 包含了将在一起显示或处理的多个轨道，每个轨道均包含（但不限于）音频、视频、文本、可隐藏字幕和字幕。 Asset 提供了关于资源的完整信息，比如时长、标题，以及渲染时的提示、视频源尺寸等等。 资产还可以包含由 `AVMetadataItem` 表示的元数据。


轨道用 `AVAssetTrack` 的实例表示，如下图所示。在典型的简单情况下，一个轨道表示音频分量，另一个表示视频分量; 在复杂的组合中，音频和视频可能有多个重叠的轨道。


![image](../../images/ios-avfoundation/avasset_and_tracks.png)


轨道具有许多属性，例如其类型（视频或音频），视频和/或音频特征，元数据和时间轴。轨道有一个数组用来描述其格式。该数组包含一组 `CMFormatDescription` 对象，每个对象描述轨道对应的媒体样本的格式。包含统一媒体的轨道（比如，都使用相同的设置来编码）将提供一个计数为 1 的数组。

轨道本身可以被划分成多段，由 `AVAssetTrackSegment` 的实例表示。段是从源到资产轨道时间轴的时间映射。


## 时间的表示

AVFoundation 中用来表示时间的数据结构主要来自于 Core Media 框架。


### 用 CMTime 表示时长

`CMTime` 是一个 C 的数据结构，它表示时间为有理数，包含分子和分母。分子表示时间值，分母表示时间刻度，分子除以分母则表示时间，单位为秒。比如当时间刻度是 10，则表示每个单位的时间值表示 1/10 秒。最常用的时间刻度是 600，因为在场景的场景中，我们用 24 fps 的电影，30 fps 的 NTSC，25 fps 的 PAL。使用 600 的时间刻度，可以准确地表示这些系统中的任何数量的帧。


除了简单的时间值之外，`CMTime` 结构可以表示非数值的值：正无穷大，负无穷大和无定义。 它也可以指示时间是否在某一点被舍入，并且它能保持一个纪元数字。


### 使用 CMTime

下面是一些使用 `CMTime` 示例：

```
CMTime time1 = CMTimeMake(200, 2); // 200 half-seconds
CMTime time2 = CMTimeMake(400, 4); // 400 quarter-seconds
 
// time1 and time2 both represent 100 seconds, but using different timescales.
if (CMTimeCompare(time1, time2) == 0) {
    NSLog(@"time1 and time2 are the same");
}
 
Float64 float64Seconds = 200.0 / 3;
CMTime time3 = CMTimeMakeWithSeconds(float64Seconds , 3); // 66.66... third-seconds
time3 = CMTimeMultiply(time3, 3);
// time3 now represents 200 seconds; next subtract time1 (100 seconds).
time3 = CMTimeSubtract(time3, time1);
CMTimeShow(time3);
 
if (CMTIME_COMPARE_INLINE(time2, ==, time3)) {
    NSLog(@"time2 and time3 are the same");
}
```

### CMTime 的特殊值

Core Media 提供了一些关于 `CMTime` 的特殊值，比如：`kCMTimeZero`，`kCMTimeInvalid`，`kCMTimePositiveInfinity`，`kCMTimeNegativeInfinity`。要检查一个表示非数值的 `CMTime` 是否是合法的，可以用 `CMTIME_IS_INVALID`，`CMTIME_IS_POSITIVE_INFINITY`，`CMTIME_IS_INDEFINITE` 这些宏。


```
CMTime myTime = <#Get a CMTime#>;
if (CMTIME_IS_INVALID(myTime)) {
    // Perhaps treat this as an error; display a suitable alert to the user.
}
```

不要去拿一个 `CMTime` 和 `kCMTimeInvalid` 做比较。


### 用对象的方式使用 CMTime


如果要使用 Core Foundation 中的一些容器来存储 `CMTime`，我们需要进行 `CMTime` 和 `CFDictionary` 之间的转换。这时我们需要用到 `CMTimeCopyAsDictionary` 和 `CMTimeMakeFromDictionary` 这些方法。我们还能用 `CMTimeCopyDescription` 来输出描述 `CMTime` 的字符串。


### 纪元


通常 `CMTime` 中的 `epoch` 设置为 0，不过我们也可以设置它为其他值来区分不相关的时间轴。比如，我们可以在循环遍历中递增 `epoch` 的值，每一个 0 到 N 的循序递增一下 `epoch` 来区分不同的轮回。


## 用 CMTimeRange 来表示时间范围


`CMTimeRange` 表示的时间范围包含两个字段：开始时间（start）和时长（duration），这两个字段都是 `CMTime` 类型。需要注意的是一个 `CMTimeRange` 不含开始时间加上时长算出来的那个时间点，即 `[start, start + duration)`。

我们可以用 `CMTimeRangeMake` 和 `CMTimeRangeFromTimeToTime` 来创建 `CMTimeRange`。但是这里有一些限制：

- `CMTimeRange` 不能跨越不同的纪元。
- `CMTime` 中的 `epoch` 字段可能不为 0，但是我们只能对 `start` 字段具有相同 `epoch` 的 `CMTimeRange` 执行相关操作（比如 `CMTimeRangeGetUnion`）。
- 表示 `duration` 的 `CMTime` 结构中的 `epoch` 应始终为 0，`value` 必须为非负数。


### 使用 CMTimeRange

Core Media 提供了一系列操作 `CMTimeRange` 的方法，比如 `CMTimeRangeContainsTime`、`CMTimeRangeEqual`、`CMTimeRangeContainsTimeRange`、`CMTimeRangeGetUnion`.

下面的代码返回值永远为 `false`：

```
CMTimeRangeContainsTime(range, CMTimeRangeGetEnd(range))
```


### CMTimeRange 的特殊值

Core Media 提供了 `kCMTimeRangeZero` 和 `kCMTimeRangeInvalid` 表示零长度 range 和错误 range。在很多情况下，`CMTimeRange` 结构可能无效，或者是零或不定式（如果其中一个 `CMTime` 字段是不确定的）。如果要测试 `CMTimeRange` 结构是否有效、零或不确定，可以使用一个适当的宏：`CMTIMERANGE_IS_VALID`，`CMTIMERANGE_IS_INVALID`，`CMTIMERANGE_IS_EMPTY` 或 `CMTIMERANGE_IS_EMPTY`。

```
CMTimeRange myTimeRange = <#Get a CMTimeRange#>;
if (CMTIMERANGE_IS_EMPTY(myTimeRange)) {
    // The time range is zero.
}
```

不要拿任何 `CMTimeRange` 和 `kCMTimeRangeInvalid` 做比较。



### 用对象的方式使用 CMTimeRange


如果要在 Core Foundation 提供的容器中使用 `CMTimeRange` 结构，则可以分别使用 `CMTimeRangeCopyAsDictionary` 和 `CMTimeRangeMakeFromDictionary` 将 `CMTimeRange` 结构转换为 `CFDictionary` 类型。 还可以使用 `CMTimeRangeCopyDescription` 函数获取 `CMTimeRange` 结构的字符串表示形式。



## 媒体的表示


视频数据及其关联的元数据在 AVFoundation 中由 Core Media 框架中的对象表示。 Core Media 使用 `CMSampleBuffer` 表示视频数据。`CMSampleBuffer` 是一种 Core Foundation 风格的类型。`CMSampleBuffer` 的一个实例在对应的 Core Video pixel buffer 中包含了视频帧的数据（参见 CVPixelBufferRef）。 我们可以使用 `CMSampleBufferGetImageBuffer` 从样本缓冲区访问 pixel buffer：

```
CVPixelBufferRef pixelBuffer = CMSampleBufferGetImageBuffer(<#A CMSampleBuffer#>);
```

从 pixel buffer 中，我们可以访问实际的视频数据。

除了视频数据之外，您还可以检索视频帧的其他方面的数据：

- 时间信息。我们可以分别使用 `CMSampleBufferGetPresentationTimeStamp` 和 `CMSampleBufferGetDecodeTimeStamp` 获得原始演示时间和解码时间的准确时间戳。
- 格式信息。格式信息封装在 `CMFormatDescription` 对象中。从格式描述中，可以分别使用 `CMVideoFormatDescriptionGetCodecType` 和 `CMVideoFormatDescriptionGetDimensions` 获取像素类型和视频尺寸。
- 元数据。元数据作为附件存储在字典中。您使用 `CMGetAttachment` 检索字典。如下面代码所示：

```
CMSampleBufferRef sampleBuffer = <#Get a sample buffer#>;
CFDictionaryRef metadataDictionary = CMGetAttachment(sampleBuffer, CFSTR("MetadataDictionary", NULL);
if (metadataDictionary) {
    // Do something with the metadata.
}
```

## 将 CMSampleBuffer 转换为 UIImage


以下代码显示如何将 `CMSampleBuffer` 转换为 `UIImage` 对象。 使用前，我们要仔细考虑对应的需求。 执行转换是比较昂贵的操作。例如，从每隔一秒钟拍摄的视频数据帧创建静止图像是合适的，但不应该使用它来实时操纵来自录制设备的每一帧视频。


```
// Create a UIImage from sample buffer data.
- (UIImage *)imageFromSampleBuffer:(CMSampleBufferRef)sampleBuffer {
    // Get a CMSampleBuffer's Core Video image buffer for the media data
    CVImageBufferRef imageBuffer = CMSampleBufferGetImageBuffer(sampleBuffer);
    // Lock the base address of the pixel buffer
    CVPixelBufferLockBaseAddress(imageBuffer, 0);
 
    // Get the number of bytes per row for the pixel buffer
    void *baseAddress = CVPixelBufferGetBaseAddress(imageBuffer);
 
    // Get the number of bytes per row for the pixel buffer
    size_t bytesPerRow = CVPixelBufferGetBytesPerRow(imageBuffer);
    // Get the pixel buffer width and height
    size_t width = CVPixelBufferGetWidth(imageBuffer);
    size_t height = CVPixelBufferGetHeight(imageBuffer);
 
    // Create a device-dependent RGB color space
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
 
    // Create a bitmap graphics context with the sample buffer data
    CGContextRef context = CGBitmapContextCreate(baseAddress, width, height, 8,
      bytesPerRow, colorSpace, kCGBitmapByteOrder32Little | kCGImageAlphaPremultipliedFirst);
    // Create a Quartz image from the pixel data in the bitmap graphics context
    CGImageRef quartzImage = CGBitmapContextCreateImage(context);
    // Unlock the pixel buffer
    CVPixelBufferUnlockBaseAddress(imageBuffer,0);
 
    // Free up the context and color space
    CGContextRelease(context);
    CGColorSpaceRelease(colorSpace);
 
    // Create an image object from the Quartz image
    UIImage *image = [UIImage imageWithCGImage:quartzImage];
 
    // Release the Quartz image
    CGImageRelease(quartzImage);
 
    return (image);
}
```






[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-av-asset
[3]: https://developer.apple.com/library/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html