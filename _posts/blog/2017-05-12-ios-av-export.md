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


### 创建 Asset Writer

为了创建 asset writer，我们需要指定输出文件的 URL 和所需的文件类型。以下代码展示了如何初始化 asset writer 来创建 QuickTime 格式的视频：


```
NSError *outError;
NSURL *outputURL = <#NSURL object representing the URL where you want to save the video#>;
AVAssetWriter *assetWriter = [AVAssetWriter assetWriterWithURL:outputURL fileType:AVFileTypeQuickTimeMovie error:&outError];
BOOL success = (assetWriter != nil);
```


### 创建 Asset Writer Inputs


想要用 asset writer 来写媒体数据，必须至少设置一个 asset writer input。例如，如果数据源已经使用 `CMSampleBufferRef` 类来表示，那么使用 `AVAssetWriterInput` 类即可。

下面的代码展示了如何设置将音频媒体数据压缩为 128 kbps AAC 并将其连接到 asset writer 的 input：

```
// Configure the channel layout as stereo.
AudioChannelLayout stereoChannelLayout = {
    .mChannelLayoutTag = kAudioChannelLayoutTag_Stereo,
    .mChannelBitmap = 0,
    .mNumberChannelDescriptions = 0
};
 
// Convert the channel layout object to an NSData object.
NSData *channelLayoutAsData = [NSData dataWithBytes:&stereoChannelLayout length:offsetof(AudioChannelLayout, mChannelDescriptions)];
 
// Get the compression settings for 128 kbps AAC.
NSDictionary *compressionAudioSettings = @{
    AVFormatIDKey         : [NSNumber numberWithUnsignedInt:kAudioFormatMPEG4AAC],
    AVEncoderBitRateKey   : [NSNumber numberWithInteger:128000],
    AVSampleRateKey       : [NSNumber numberWithInteger:44100],
    AVChannelLayoutKey    : channelLayoutAsData,
    AVNumberOfChannelsKey : [NSNumber numberWithUnsignedInteger:2]
};
 
// Create the asset writer input with the compression settings and specify the media type as audio.
AVAssetWriterInput *assetWriterInput = [AVAssetWriterInput assetWriterInputWithMediaType:AVMediaTypeAudio outputSettings:compressionAudioSettings];
// Add the input to the writer if possible.
if ([assetWriter canAddInput:assetWriterInput]) {
    [assetWriter addInput:assetWriterInput];
}
```

需要注意的是，如果要以原本存储的格式来写入媒体数据，可以在 `outputSettings` 参数中传 `nil`。只有 asset writer 使用 `AVFileTypeQuickTimeMovie` 的 `fileType` 进行初始化时，才能传 `nil`。


您的 asset writer input 可以通过设置 `metadata` 和 `transform` 属性来包含一些元数据或为特定的轨道指定不同的变换。对于数据源是视频轨道的 asset writer input，您可以通过执行以下操作来将视频的原始变换保存在输出文件中：


```
AVAsset *videoAsset = <#AVAsset with at least one video track#>;
AVAssetTrack *videoAssetTrack = [[videoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
assetWriterInput.transform = videoAssetTrack.preferredTransform;
```

需要注意的是 `metadata` 和 `transform` 这两个属性需要在开始写之前设定才会生效。


将媒体数据写入输出文件时，有时我们可能需要分配像素数据缓冲区。这时可以使用 `AVAssetWriterInputPixelBufferAdaptor` 类。为了最大的效率，不要使用单独的池分配的像素缓冲区，而是使用像素缓冲适配器提供的像素缓冲池。以下代码创建一个在 RGB 域中工作的像素缓冲区对象，它将使用 `CGImage` 对象来创建其像素缓冲区：


```
NSDictionary *pixelBufferAttributes = @{
     kCVPixelBufferCGImageCompatibilityKey: [NSNumber numberWithBool:YES],
     kCVPixelBufferCGBitmapContextCompatibilityKey: [NSNumber numberWithBool:YES],
     kCVPixelBufferPixelFormatTypeKey: [NSNumber numberWithInt:kCVPixelFormatType_32ARGB]
};
AVAssetWriterInputPixelBufferAdaptor *inputPixelBufferAdaptor = [AVAssetWriterInputPixelBufferAdaptor assetWriterInputPixelBufferAdaptorWithAssetWriterInput:self.assetWriterInput sourcePixelBufferAttributes:pixelBufferAttributes];
```


需要注意的是，所有 `AVAssetWriterInputPixelBufferAdaptor` 对象必须连接到单个 asset writer input。Asset writer input 必须接受 `AVMediaTypeVideo` 类型的媒体数据。



### 写媒体数据


配置 asset writer 所需的所有输入后，即可开始写媒体数据。与 asset reader 一样，通过调用 `startWriting` 方法启动写入过程。然后，需要通过调用 `startSessionAtSourceTime:` 方法启动写入会话。Asset writer 完成的所有写入都必须在这些会话之一内进行，每个会话的时间范围不超出源中包含的媒体数据的时间范围。

下面的代码展示了当我们的数据源是一个 asset reader，它提供从 `AVAsset` 对象读取的媒体数据，并且不希望包含资产前半部分的媒体数据：


```
CMTime halfAssetDuration = CMTimeMultiplyByFloat64(self.asset.duration, 0.5);
[self.assetWriter startSessionAtSourceTime:halfAssetDuration];
//Implementation continues.
```

通常，要结束写入会话，您必须调用 `endSessionAtSourceTime:` 方法。但是，如果写入会话直到文件的末尾，则可以通过调用 `finishWriting` 方法来结束。

下面代码展示了使用单个 input 启动 asset writer 并写入其所有媒体数据：


```
// Prepare the asset writer for writing.
[self.assetWriter startWriting];
// Start a sample-writing session.
[self.assetWriter startSessionAtSourceTime:kCMTimeZero];
// Specify the block to execute when the asset writer is ready for media data and the queue to call it on.
[self.assetWriterInput requestMediaDataWhenReadyOnQueue:myInputSerialQueue usingBlock:^{
    while ([self.assetWriterInput isReadyForMoreMediaData]) {
        // Get the next sample buffer.
        CMSampleBufferRef nextSampleBuffer = [self copyNextSampleBufferToWrite];
        if (nextSampleBuffer) {
            // If it exists, append the next sample buffer to the output file.
            [self.assetWriterInput appendSampleBuffer:nextSampleBuffer];
            CFRelease(nextSampleBuffer);
            nextSampleBuffer = nil;
        } else {
            // Assume that lack of a next sample buffer means the sample buffer source is out of samples and mark the input as finished.
            [self.assetWriterInput markAsFinished];
            break;
        }
    }
}];
```

上面代码中的 `copyNextSampleBufferToWrite` 方法只是一个占位方法。在这个方法里需要实现一些逻辑来返回将要写入的媒体数据，用 `CMSampleBufferRef` 对象表示，这里的 sample buffer 就可以用 asset reader input 作为数据来源。


## 重编码 Asset

您可以一起使用 asset reader 和 asset writer 对象将 asset 从一种格式转换为另一种。使用这些对象，我们对转换的控制比 `AVAssetExportSession` 要多得多。比如，我们可以选择哪些轨道要写入到输出文件中；指定输出格式；在转换过程中修改 asset。此过程的第一步只是根据需要设置 asset reader outputs 和 asset writer inputs。在 asset reader 完全配置后，可以分别调用 `startReading` 和 `startWriting` 方法来启动。

以下代码展示了如何使用单个 asset writer input 来写入由单个 asset reader output 提供的媒体数据：


```
NSString *serializationQueueDescription = [NSString stringWithFormat:@"%@ serialization queue", self];
 
// Create a serialization queue for reading and writing.
dispatch_queue_t serializationQueue = dispatch_queue_create([serializationQueueDescription UTF8String], NULL);
 
// Specify the block to execute when the asset writer is ready for media data and the queue to call it on.
[self.assetWriterInput requestMediaDataWhenReadyOnQueue:serializationQueue usingBlock:^{
    while ([self.assetWriterInput isReadyForMoreMediaData]) {
        // Get the asset reader output's next sample buffer.
        CMSampleBufferRef sampleBuffer = [self.assetReaderOutput copyNextSampleBuffer];
        if (sampleBuffer != NULL) {
            // If it exists, append this sample buffer to the output file.
            BOOL success = [self.assetWriterInput appendSampleBuffer:sampleBuffer];
            CFRelease(sampleBuffer);
            sampleBuffer = NULL;
            // Check for errors that may have occurred when appending the new sample buffer.
            if (!success && self.assetWriter.status == AVAssetWriterStatusFailed) {
                NSError *failureError = self.assetWriter.error;
                //Handle the error.
            }
        } else {
            // If the next sample buffer doesn't exist, find out why the asset reader output couldn't vend another one.
            if (self.assetReader.status == AVAssetReaderStatusFailed) {
                NSError *failureError = self.assetReader.error;
                //Handle the error here.
            } else {
                // The asset reader output must have vended all of its samples. Mark the input as finished.
                [self.assetWriterInput markAsFinished];
                break;
            }
        }
    }
}];
```


## 一个完整示例


下面的代码展示了如何使用 asset reader 和 asset writer 将资产的第一个视频轨道和音频轨重新编码到新的文件中。其中主要包括下面这些步骤：


- 使用序列化队列来处理读和写视听数据。
- 初始化 asset reader 并配置两个 output，一个用于音频，另一个用于视频。
- 初始化 asset writer 并配置两个 input，一个用于音频，另一个用于视频。
- 使用 asset reader 通过两种不同的输出/输入组合异步地向 asset writer 提供媒体数据。
- 使用 dispatch group 来完成重编码的异步调度。
- 允许用户在编码开始后取消该操作。

下面是初始化的代码：

```
// 初始化过程：
NSString *serializationQueueDescription = [NSString stringWithFormat:@"%@ serialization queue", self];
 
// Create the main serialization queue.
self.mainSerializationQueue = dispatch_queue_create([serializationQueueDescription UTF8String], NULL);
NSString *rwAudioSerializationQueueDescription = [NSString stringWithFormat:@"%@ rw audio serialization queue", self];
 
// Create the serialization queue to use for reading and writing the audio data.
self.rwAudioSerializationQueue = dispatch_queue_create([rwAudioSerializationQueueDescription UTF8String], NULL);
NSString *rwVideoSerializationQueueDescription = [NSString stringWithFormat:@"%@ rw video serialization queue", self];
 
// Create the serialization queue to use for reading and writing the video data.
self.rwVideoSerializationQueue = dispatch_queue_create([rwVideoSerializationQueueDescription UTF8String], NULL);


// 加载资源：
self.asset = <#AVAsset that you want to reencode#>;
self.cancelled = NO;
self.outputURL = <#NSURL representing desired output URL for file generated by asset writer#>;
// Asynchronously load the tracks of the asset you want to read.
[self.asset loadValuesAsynchronouslyForKeys:@[@"tracks"] completionHandler:^{
    // Once the tracks have finished loading, dispatch the work to the main serialization queue.
    dispatch_async(self.mainSerializationQueue, ^{
        // Due to asynchronous nature, check to see if user has already cancelled.
        if (self.cancelled) {
            return;
        }
        BOOL success = YES;
        NSError *localError = nil;
        // Check for success of loading the assets tracks.
        success = ([self.asset statusOfValueForKey:@"tracks" error:&localError] == AVKeyValueStatusLoaded);
        if (success) {
            // If the tracks loaded successfully, make sure that no file exists at the output path for the asset writer.
            NSFileManager *fm = [NSFileManager defaultManager];
            NSString *localOutputPath = [self.outputURL path];
            if ([fm fileExistsAtPath:localOutputPath]) {
                success = [fm removeItemAtPath:localOutputPath error:&localError];
            }
        }
        if (success) {
            success = [self setupAssetReaderAndAssetWriter:&localError];
        }
        if (success) {
            success = [self startAssetReaderAndWriter:&localError];
        }
        if (!success) {
            [self readingAndWritingDidFinishSuccessfully:success withError:localError];
        }
    });
}];

```


下面是初始化 asset reader 和 writer 的代码：


```
- (BOOL)setupAssetReaderAndAssetWriter:(NSError **)outError {
     // Create and initialize the asset reader.
     self.assetReader = [[AVAssetReader alloc] initWithAsset:self.asset error:outError];
     BOOL success = (self.assetReader != nil);
     if (success) {
          // If the asset reader was successfully initialized, do the same for the asset writer.
          self.assetWriter = [[AVAssetWriter alloc] initWithURL:self.outputURL fileType:AVFileTypeQuickTimeMovie error:outError];
          success = (self.assetWriter != nil);
     }
 
     if (success) {
          // If the reader and writer were successfully initialized, grab the audio and video asset tracks that will be used.
          AVAssetTrack *assetAudioTrack = nil, *assetVideoTrack = nil;
          NSArray *audioTracks = [self.asset tracksWithMediaType:AVMediaTypeAudio];
          if ([audioTracks count] > 0) {
               assetAudioTrack = [audioTracks objectAtIndex:0];
          }
          NSArray *videoTracks = [self.asset tracksWithMediaType:AVMediaTypeVideo];
          if ([videoTracks count] > 0) {
               assetVideoTrack = [videoTracks objectAtIndex:0];
          }
 
          if (assetAudioTrack) {
               // If there is an audio track to read, set the decompression settings to Linear PCM and create the asset reader output.
               NSDictionary *decompressionAudioSettings = @{ AVFormatIDKey : [NSNumber numberWithUnsignedInt:kAudioFormatLinearPCM] };
               self.assetReaderAudioOutput = [AVAssetReaderTrackOutput assetReaderTrackOutputWithTrack:assetAudioTrack outputSettings:decompressionAudioSettings];
               [self.assetReader addOutput:self.assetReaderAudioOutput];
               // Then, set the compression settings to 128kbps AAC and create the asset writer input.
               AudioChannelLayout stereoChannelLayout = {
                    .mChannelLayoutTag = kAudioChannelLayoutTag_Stereo,
                    .mChannelBitmap = 0,
                    .mNumberChannelDescriptions = 0
               };
               NSData *channelLayoutAsData = [NSData dataWithBytes:&stereoChannelLayout length:offsetof(AudioChannelLayout, mChannelDescriptions)];
               NSDictionary *compressionAudioSettings = @{
                    AVFormatIDKey         : [NSNumber numberWithUnsignedInt:kAudioFormatMPEG4AAC],
                    AVEncoderBitRateKey   : [NSNumber numberWithInteger:128000],
                    AVSampleRateKey       : [NSNumber numberWithInteger:44100],
                    AVChannelLayoutKey    : channelLayoutAsData,
                    AVNumberOfChannelsKey : [NSNumber numberWithUnsignedInteger:2]
               };
               self.assetWriterAudioInput = [AVAssetWriterInput assetWriterInputWithMediaType:[assetAudioTrack mediaType] outputSettings:compressionAudioSettings];
               [self.assetWriter addInput:self.assetWriterAudioInput];
          }
 
          if (assetVideoTrack) {
               // If there is a video track to read, set the decompression settings for YUV and create the asset reader output.
               NSDictionary *decompressionVideoSettings = @{
                    (id)kCVPixelBufferPixelFormatTypeKey     : [NSNumber numberWithUnsignedInt:kCVPixelFormatType_422YpCbCr8],
                    (id)kCVPixelBufferIOSurfacePropertiesKey : [NSDictionary dictionary]
               };
               self.assetReaderVideoOutput = [AVAssetReaderTrackOutput assetReaderTrackOutputWithTrack:assetVideoTrack outputSettings:decompressionVideoSettings];
               [self.assetReader addOutput:self.assetReaderVideoOutput];
               CMFormatDescriptionRef formatDescription = NULL;
               // Grab the video format descriptions from the video track and grab the first one if it exists.
               NSArray *videoFormatDescriptions = [assetVideoTrack formatDescriptions];
               if ([videoFormatDescriptions count] > 0) {
                    formatDescription = (__bridge CMFormatDescriptionRef)[formatDescriptions objectAtIndex:0];
               }
               CGSize trackDimensions = {
                    .width = 0.0,
                    .height = 0.0,
               };
               // If the video track had a format description, grab the track dimensions from there. Otherwise, grab them direcly from the track itself.
               if (formatDescription) {
                    trackDimensions = CMVideoFormatDescriptionGetPresentationDimensions(formatDescription, false, false);
               } else {
                    trackDimensions = [assetVideoTrack naturalSize];
               }
               NSDictionary *compressionSettings = nil;
               // If the video track had a format description, attempt to grab the clean aperture settings and pixel aspect ratio used by the video.
               if (formatDescription) {
                    NSDictionary *cleanAperture = nil;
                    NSDictionary *pixelAspectRatio = nil;
                    CFDictionaryRef cleanApertureFromCMFormatDescription = CMFormatDescriptionGetExtension(formatDescription, kCMFormatDescriptionExtension_CleanAperture);
                    if (cleanApertureFromCMFormatDescription) {
                         cleanAperture = @{
                              AVVideoCleanApertureWidthKey            : (id)CFDictionaryGetValue(cleanApertureFromCMFormatDescription, kCMFormatDescriptionKey_CleanApertureWidth),
                              AVVideoCleanApertureHeightKey           : (id)CFDictionaryGetValue(cleanApertureFromCMFormatDescription, kCMFormatDescriptionKey_CleanApertureHeight),
                              AVVideoCleanApertureHorizontalOffsetKey : (id)CFDictionaryGetValue(cleanApertureFromCMFormatDescription, kCMFormatDescriptionKey_CleanApertureHorizontalOffset),
                              AVVideoCleanApertureVerticalOffsetKey   : (id)CFDictionaryGetValue(cleanApertureFromCMFormatDescription, kCMFormatDescriptionKey_CleanApertureVerticalOffset)
                         };
                    }
                    CFDictionaryRef pixelAspectRatioFromCMFormatDescription = CMFormatDescriptionGetExtension(formatDescription, kCMFormatDescriptionExtension_PixelAspectRatio);
                    if (pixelAspectRatioFromCMFormatDescription) {
                         pixelAspectRatio = @{
                              AVVideoPixelAspectRatioHorizontalSpacingKey : (id)CFDictionaryGetValue(pixelAspectRatioFromCMFormatDescription, kCMFormatDescriptionKey_PixelAspectRatioHorizontalSpacing),
                              AVVideoPixelAspectRatioVerticalSpacingKey   : (id)CFDictionaryGetValue(pixelAspectRatioFromCMFormatDescription, kCMFormatDescriptionKey_PixelAspectRatioVerticalSpacing)
                         };
                    }
                    // Add whichever settings we could grab from the format description to the compression settings dictionary.
                    if (cleanAperture || pixelAspectRatio) {
                         NSMutableDictionary *mutableCompressionSettings = [NSMutableDictionary dictionary];
                         if (cleanAperture) {
                              [mutableCompressionSettings setObject:cleanAperture forKey:AVVideoCleanApertureKey];
                         }
                         if (pixelAspectRatio) {
                              [mutableCompressionSettings setObject:pixelAspectRatio forKey:AVVideoPixelAspectRatioKey];
                         }
                         compressionSettings = mutableCompressionSettings;
                    }
               }
               // Create the video settings dictionary for H.264.
               NSMutableDictionary *videoSettings = (NSMutableDictionary *) @{
                    AVVideoCodecKey  : AVVideoCodecH264,
                    AVVideoWidthKey  : [NSNumber numberWithDouble:trackDimensions.width],
                    AVVideoHeightKey : [NSNumber numberWithDouble:trackDimensions.height]
               };
               // Put the compression settings into the video settings dictionary if we were able to grab them.
               if (compressionSettings) {
                    [videoSettings setObject:compressionSettings forKey:AVVideoCompressionPropertiesKey];
               }
               // Create the asset writer input and add it to the asset writer.
               self.assetWriterVideoInput = [AVAssetWriterInput assetWriterInputWithMediaType:[videoTrack mediaType] outputSettings:videoSettings];
               [self.assetWriter addInput:self.assetWriterVideoInput];
          }
     }
     return success;
}

```

下面是重新编码 asset 的代码：


```
- (BOOL)startAssetReaderAndWriter:(NSError **)outError {
     BOOL success = YES;
     // Attempt to start the asset reader.
     success = [self.assetReader startReading];
     if (!success) {
          *outError = [self.assetReader error];
     }
     if (success) {
          // If the reader started successfully, attempt to start the asset writer.
          success = [self.assetWriter startWriting];
          if (!success) {
               *outError = [self.assetWriter error];
          }
     }
 
     if (success) {
          // If the asset reader and writer both started successfully, create the dispatch group where the reencoding will take place and start a sample-writing session.
          self.dispatchGroup = dispatch_group_create();
          [self.assetWriter startSessionAtSourceTime:kCMTimeZero];
          self.audioFinished = NO;
          self.videoFinished = NO;
 
          if (self.assetWriterAudioInput) {
               // If there is audio to reencode, enter the dispatch group before beginning the work.
               dispatch_group_enter(self.dispatchGroup);
               // Specify the block to execute when the asset writer is ready for audio media data, and specify the queue to call it on.
               [self.assetWriterAudioInput requestMediaDataWhenReadyOnQueue:self.rwAudioSerializationQueue usingBlock:^{
                    // Because the block is called asynchronously, check to see whether its task is complete.
                    if (self.audioFinished) {
                         return;
                     }
                    BOOL completedOrFailed = NO;
                    // If the task isn't complete yet, make sure that the input is actually ready for more media data.
                    while ([self.assetWriterAudioInput isReadyForMoreMediaData] && !completedOrFailed) {
                         // Get the next audio sample buffer, and append it to the output file.
                         CMSampleBufferRef sampleBuffer = [self.assetReaderAudioOutput copyNextSampleBuffer];
                         if (sampleBuffer != NULL) {
                              BOOL success = [self.assetWriterAudioInput appendSampleBuffer:sampleBuffer];
                              CFRelease(sampleBuffer);
                              sampleBuffer = NULL;
                              completedOrFailed = !success;
                         } else {
                              completedOrFailed = YES;
                         }
                    }
                    if (completedOrFailed) {
                         // Mark the input as finished, but only if we haven't already done so, and then leave the dispatch group (since the audio work has finished).
                         BOOL oldFinished = self.audioFinished;
                         self.audioFinished = YES;
                         if (oldFinished == NO) {
                              [self.assetWriterAudioInput markAsFinished];
                         }
                         dispatch_group_leave(self.dispatchGroup);
                    }
               }];
          }
 
          if (self.assetWriterVideoInput) {
               // If we had video to reencode, enter the dispatch group before beginning the work.
               dispatch_group_enter(self.dispatchGroup);
               // Specify the block to execute when the asset writer is ready for video media data, and specify the queue to call it on.
               [self.assetWriterVideoInput requestMediaDataWhenReadyOnQueue:self.rwVideoSerializationQueue usingBlock:^{
                    // Because the block is called asynchronously, check to see whether its task is complete.
                    if (self.videoFinished) {
                         return;
                     }
                    BOOL completedOrFailed = NO;
                    // If the task isn't complete yet, make sure that the input is actually ready for more media data.
                    while ([self.assetWriterVideoInput isReadyForMoreMediaData] && !completedOrFailed) {
                         // Get the next video sample buffer, and append it to the output file.
                         CMSampleBufferRef sampleBuffer = [self.assetReaderVideoOutput copyNextSampleBuffer];
                         if (sampleBuffer != NULL) {
                              BOOL success = [self.assetWriterVideoInput appendSampleBuffer:sampleBuffer];
                              CFRelease(sampleBuffer);
                              sampleBuffer = NULL;
                              completedOrFailed = !success;
                         } else {
                              completedOrFailed = YES;
                         }
                    }
                    if (completedOrFailed) {
                         // Mark the input as finished, but only if we haven't already done so, and then leave the dispatch group (since the video work has finished).
                         BOOL oldFinished = self.videoFinished;
                         self.videoFinished = YES;
                         if (oldFinished == NO) {
                              [self.assetWriterVideoInput markAsFinished];
                         }
                         dispatch_group_leave(self.dispatchGroup);
                    }
               }];
          }
          // Set up the notification that the dispatch group will send when the audio and video work have both finished.
          dispatch_group_notify(self.dispatchGroup, self.mainSerializationQueue, ^{
               BOOL finalSuccess = YES;
               NSError *finalError = nil;
               // Check to see if the work has finished due to cancellation.
               if (self.cancelled) {
                    // If so, cancel the reader and writer.
                    [self.assetReader cancelReading];
                    [self.assetWriter cancelWriting];
               } else {
                    // If cancellation didn't occur, first make sure that the asset reader didn't fail.
                    if ([self.assetReader status] == AVAssetReaderStatusFailed) {
                         finalSuccess = NO;
                         finalError = [self.assetReader error];
                    }
                    // If the asset reader didn't fail, attempt to stop the asset writer and check for any errors.
                    if (finalSuccess) {
                         finalSuccess = [self.assetWriter finishWriting];
                         if (!finalSuccess) {
                              finalError = [self.assetWriter error];
                         }
                    }
               }
               // Call the method to handle completion, and pass in the appropriate parameters to indicate whether reencoding was successful.
               [self readingAndWritingDidFinishSuccessfully:finalSuccess withError:finalError];
          });
     }
     // Return success here to indicate whether the asset reader and writer were started successfully.
     return success;
}
```


处理完成的代码：

```
- (void)readingAndWritingDidFinishSuccessfully:(BOOL)success withError:(NSError *)error {
     if (!success) {
          // If the reencoding process failed, we need to cancel the asset reader and writer.
          [self.assetReader cancelReading];
          [self.assetWriter cancelWriting];
          dispatch_async(dispatch_get_main_queue(), ^{
               // Handle any UI tasks here related to failure.
          });
     } else {
          // Reencoding was successful, reset booleans.
          self.cancelled = NO;
          self.videoFinished = NO;
          self.audioFinished = NO;
          dispatch_async(dispatch_get_main_queue(), ^{
               // Handle any UI tasks here related to success.
          });
     }
}
```


处理取消的代码：


```
- (void)cancel
{
     // Handle cancellation asynchronously, but serialize it with the main queue.
     dispatch_async(self.mainSerializationQueue, ^{
          // If we had audio data to reencode, we need to cancel the audio work.
          if (self.assetWriterAudioInput) {
               // Handle cancellation asynchronously again, but this time serialize it with the audio queue.
               dispatch_async(self.rwAudioSerializationQueue, ^{
                    // Update the Boolean property indicating the task is complete and mark the input as finished if it hasn't already been marked as such.
                    BOOL oldFinished = self.audioFinished;
                    self.audioFinished = YES;
                    if (oldFinished == NO) {
                         [self.assetWriterAudioInput markAsFinished];
                    }
                    // Leave the dispatch group since the audio work is finished now.
                    dispatch_group_leave(self.dispatchGroup);
               });
          }
 
          if (self.assetWriterVideoInput) {
               // Handle cancellation asynchronously again, but this time serialize it with the video queue.
               dispatch_async(self.rwVideoSerializationQueue, ^{
                    // Update the Boolean property indicating the task is complete and mark the input as finished if it hasn't already been marked as such.
                    BOOL oldFinished = self.videoFinished;
                    self.videoFinished = YES;
                    if (oldFinished == NO) {
                         [self.assetWriterVideoInput markAsFinished];
                    }
                    // Leave the dispatch group, since the video work is finished now.
                    dispatch_group_leave(self.dispatchGroup);
               });
          }
          // Set the cancelled Boolean property to YES to cancel any work on the main queue as well.
          self.cancelled = YES;
     });
}
```


## Asset Output 设置助手


`AVOutputSettingsAssistant` 类有助于为 asset reader 或 writer 创建输出设置字典。这使得设置更简单，特别是对于具有多个特定预设的高帧率 H264 电影。

下面的示例是如何使用 output settings assistant：

```
AVOutputSettingsAssistant *outputSettingsAssistant = [AVOutputSettingsAssistant outputSettingsAssistantWithPreset:<#some preset#>];
CMFormatDescriptionRef audioFormat = [self getAudioFormat];
 
if (audioFormat != NULL) {
    [outputSettingsAssistant setSourceAudioFormat:(CMAudioFormatDescriptionRef)audioFormat];
}
 
CMFormatDescriptionRef videoFormat = [self getVideoFormat];
 
if (videoFormat != NULL) {
    [outputSettingsAssistant setSourceVideoFormat:(CMVideoFormatDescriptionRef)videoFormat];
}
 
CMTime assetMinVideoFrameDuration = [self getMinFrameDuration];
CMTime averageFrameDuration = [self getAvgFrameDuration]
 
[outputSettingsAssistant setSourceVideoAverageFrameDuration:averageFrameDuration];
[outputSettingsAssistant setSourceVideoMinFrameDuration:assetMinVideoFrameDuration];
 
AVAssetWriter *assetWriter = [AVAssetWriter assetWriterWithURL:<#some URL#> fileType:[outputSettingsAssistant outputFileType] error:NULL];
AVAssetWriterInput *audioInput = [AVAssetWriterInput assetWriterInputWithMediaType:AVMediaTypeAudio outputSettings:[outputSettingsAssistant audioSettings] sourceFormatHint:audioFormat];
AVAssetWriterInput *videoInput = [AVAssetWriterInput assetWriterInputWithMediaType:AVMediaTypeVideo outputSettings:[outputSettingsAssistant videoSettings] sourceFormatHint:videoFormat];
```








[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-av-asset
[3]: https://developer.apple.com/library/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html