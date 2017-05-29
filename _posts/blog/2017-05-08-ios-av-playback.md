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


我们配置 asset 来播放的方式多多少少会依赖 asset 的类型，一般我们有两种不同类型的 asset：

- 1）基于文件的 asset，一般可以来源于本地视频文件、相册资源库等等。
- 2）流式 asset，比如 HLS 格式的视频。


加载基于文件的 asset 一般分为如下几步：

- 基于文件路径的 URL 创建 `AVURLAsset` 实例。
- 基于 `AVURLAsset` 实例创建 `AVPlayerItem` 实例。
- 将 `AVPlayerItem` 实例与一个 `AVPlayer` 实例关联。
- KVO 监测 `AVPlayerItem` 的 `status` 属性来等待其已经可播，即加载完成。


创建并加载一个 HTTP Live Stream（HLS）格式的资源来播放时，可以按照下面几步来做：

- 基于资源的 URL 初始化一个 `AVPlayerItem` 实例，因为你无法直接创建一个 `AVAsset` 来表示 HLS 资源。
- 当你将 `AVPlayerItem` 和 `AVPlayer` 实例关联起来后，他就开始为播放做准备，当一切就绪时 `AVPlayerItem` 会创建出  `AVAsset` 和 `AVAssetTrack` 实例以用来对接 HLS 视频流的音视频内容。
- 要获取视频流的时长，你需要 KVO 监测 `AVPlayerItem` 的 `duration` 属性，当资源可以播放时，它会被更新为正确的值。


```
NSURL *url = [NSURL URLWithString:@"<#Live stream URL#>];
// You may find a test stream at <http://devimages.apple.com/iphone/samples/bipbop/bipbopall.m3u8>.
self.playerItem = [AVPlayerItem playerItemWithURL:url];
[playerItem addObserver:self forKeyPath:@"status" options:0 context:&ItemStatusContext];
self.player = [AVPlayer playerWithPlayerItem:playerItem];
```

当你不知道一个 URL 对应的是什么类型的 asset 时，你可以这样做：

- 尝试基于 URL 来初始化一个 `AVURLAsset`，并加载它的 `tracks` 属性。如果 `tracks` 属性加载成功，就基于 asset 来创建一个 `AVPlayerItem` 实例。
- 如果 `tracks` 属性加载失败，那么就直接基于 URL 创建一个 `AVPlayerItem` 实例，并 KVO 监测 `AVPlayer` 的 `status` 属性来看它何时可以播放。
- 如果上述尝试都失败，那就清理掉 `AVPlayerItem`。


## 播放一个 AVPlayerItem

调用 `AVPlayer` 的 `play` 接口即可开始播放。

```
- (IBAction)play:sender {
    [player play];
}
```

除了简单的播放，还可以通过设置 `rate` 属性设置播放速率。

```
player.rate = 0.5;
player.rate = 2.0;
```

播放速率设置为 1.0 表示正常播放，设置为 0.0 表示暂停（等同调用 `pause` 效果）。

除了正向播放，有的音视频还能支持倒播，不过需要需要检查几个属性：

- `canPlayReverse`：支持设置播放速率为 -1.0。
- `canPlaySlowReverse`：支持设置播放速率为 -1.0 到 0.0。
- `canPlayFastReverse`：支持设置播放速率为小于 -1.0 的值。


可以通过 `seekToTime:` 接口来调整播放位置。但是这个接口主要是为性能考虑，不保证精确。

```
CMTime fiveSecondsIn = CMTimeMake(5, 1);
[player seekToTime:fiveSecondsIn];
```

如果要精确调整，可以用 `seekToTime:toleranceBefore:toleranceAfter:` 接口。

```
CMTime fiveSecondsIn = CMTimeMake(5, 1);
[player seekToTime:fiveSecondsIn toleranceBefore:kCMTimeZero toleranceAfter:kCMTimeZero];
```

需要注意的是，设置 tolerance 为 zero 会耗费较大的计算性能，所以一般只在编写复杂的音视频编辑功能是这样设置。


我们可以通过监听 `AVPlayerItemDidPlayToEndTimeNotification` 来获得播放结束事件，在播放结束后可以用 `seekToTime:` 调整播放位置到 zero，否则调用 `play` 会无效。

```
// Register with the notification center after creating the player item.
[[NSNotificationCenter defaultCenter] 
	addObserver:self 
	   selector:@selector(playerItemDidReachEnd:)
    	   name:AVPlayerItemDidPlayToEndTimeNotification
         object:<#The player item#>];
 
- (void)playerItemDidReachEnd:(NSNotification *)notification {
    [player seekToTime:kCMTimeZero];
}
```

此外，我们还能设置播放器的 `actionAtItemEnd` 属性来设置其在播放结束后的行为，比如 `AVPlayerActionAtItemEndPause` 表示播放结束后会暂停。



## 播放多个 AVPlayerItem

我们可以用 `AVQueuePlayer` 来顺序播放多个 `AVPlayerItem`。`AVQueuePlayer` 是 `AVPlayer` 的子类。

```
NSArray *items = <#An array of player items#>;
AVQueuePlayer *queuePlayer = [[AVQueuePlayer alloc] initWithItems:items];
```

通过调用 `play` 即可顺序播放，也可以调用 `advanceToNextItem` 跳到下个 item。除此之外，我们还可以用 `insertItem:afterItem:`、`removeItem:`、`removeAllItems` 来控制播放资源。

当插入一个 item 的时候，可以需要用 `canInsertItem:afterItem:` 检查下是否可以插入， 对 afterItem 传入 nil，则检查是否可以插入到队尾。


```
AVPlayerItem *anItem = <#Get a player item#>;
if ([queuePlayer canInsertItem:anItem afterItem:nil]) {
    [queuePlayer insertItem:anItem afterItem:nil];
}
```


## 监测播放状态

我们可以监测一些 `AVPlayer` 的状态和正在播放的 `AVPlayerItem` 的状态，这对于处理那些不在你直接控制下的 state 是很有用的，比如：

- 如果用户使用多任务处理切换到另一个应用程序，播放器的 rate 属性将下降到 0.0。
- 当播放远程媒体资源（比如网络视频）时，监测 `AVPlayerItem` 的 `loadedTimeRanges` 和 `seekableTimeRanges` 可以知道可以播放和 seek 的资源时长。
- 当播放 HTTP Live Stream 时，播放器的 `currentItem` 可能发生变化。
- 当播放 HTTP Live Stream 时，`AVPlayerItem` 的 `tracks` 可能发生变化。这种情况可能发生在播放流切换了编码。
- 当播放失败时，`AVPlayer` 或 `AVPlayerItem` 的 `status` 可能发生变化。




### 响应 status 属性的变化

通过 KVO 监测 `AVPlayer` 和正在播放的 `AVPlayerItem` 的 `status` 属性，可以获得对应的通知，比如当播放出现错误时，你可能会收到 `AVPlayerStatusFailed` 或 `AVPlayerItemStatusFailed` 通知，这时你就可以做相应的处理。

需要注意的是，由于 `AVFoundation` 不会指定在哪个线程发送通知，所以如果你需要在收到通知后更新用户界面的话，你需要切到主线程。


```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
 
    if (context == <#Player status context#>) {
        AVPlayer *thePlayer = (AVPlayer *) object;
        if ([thePlayer status] == AVPlayerStatusFailed) {
            NSError *error = [<#The AVPlayer object#> error];
            // Respond to error: for example, display an alert sheet.
            return;
        }
        // Deal with other status change if appropriate.
    }
    // Deal with other change notifications if appropriate.
    [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    return;
}
```

### 跟踪视觉内容就绪状态


我们可以监测 `AVPlayerLayer` 实例的 `readyForDisplay` 属性来获得播放器已经可以开始渲染视觉内容的通知。

基于这个能力，我们就能实现在播放器的视觉内容就绪时才将 player layer 插入到 layer 树中去展示给用户。


### 追踪播放时间变化


我们可以使用 `AVPlayer` 的 `addPeriodicTimeObserverForInterval:queue:usingBlock:` 和 `addBoundaryTimeObserverForTimes:queue:usingBlock:` 这两个接口来追踪当前播放位置的变化，这样我们就可以在用户界面上做出更新，反馈给用户当前的播放时间和剩余的播放时间等等。

- `addPeriodicTimeObserverForInterval:queue:usingBlock:`，这个接口将会在播放时间发生变化时在回调 block 中通知我们当前播放时间。
- `addBoundaryTimeObserverForTimes:queue:usingBlock:`，这个接口允许我们传入一组时间（CMTime 数组）当播放器播到这些时间时会在回调 block 中通知我们。


这两个接口都会返回一个 observer 角色的对象给我们，我们需要在监测时间的这个过程中强引用这个对象，同时在不需要使用它时调用 `removeTimeObserver:` 接口来移除它。

此外，AVFoundation 也不保证在每次时间变化或设置时间到达时都回调 block 来通知你。比如当上一次回调 block 还没完成的情况时，又到了此次回调 block 的时机，AVFoundation 这次就不会调用 block。所以我们需要确保不要在 block 回调里做开销太大、耗时太长的任务。


```
// Assume a property: @property (strong) id playerObserver;
 
Float64 durationSeconds = CMTimeGetSeconds([<#An asset#> duration]);
CMTime firstThird = CMTimeMakeWithSeconds(durationSeconds/3.0, 1);
CMTime secondThird = CMTimeMakeWithSeconds(durationSeconds*2.0/3.0, 1);
NSArray *times = @[[NSValue valueWithCMTime:firstThird], [NSValue valueWithCMTime:secondThird]];
 
self.playerObserver = [<#A player#> addBoundaryTimeObserverForTimes:times queue:NULL usingBlock:^{
 
    NSString *timeDescription = (NSString *)
        CFBridgingRelease(CMTimeCopyDescription(NULL, [self.player currentTime]));
    NSLog(@"Passed a boundary at %@", timeDescription);
}];
```



### 播放结束

监听 `AVPlayerItemDidPlayToEndTimeNotification` 这个通知即可。上文有提到，这里不再重复。



## 一个完整示例



这里的示例将展示如果使用 `AVPlayer` 来播放一个视频文件，主要包括下面几个步骤：

- 配置一个使用 `AVPlayerLayer` layer 的 `UIView`。
- 创建一个 `AVPlayer` 实例。
- 基于文件类型的 asset 创建一个 `AVPlayerItem` 实例，并用 KVO 监测其 `status` 属性。
- 响应 `AVPlayerItem` 实例可以播放的通知，显示出一个按钮。
- 播放 `AVPlayerItem` 并播放完成后将其播放位置调整到开始位置。


首先是 PlayerView：

```
#import <UIKit/UIKit.h>
#import <AVFoundation/AVFoundation.h>
 
@interface PlayerView : UIView
@property (nonatomic) AVPlayer *player;
@end
 
@implementation PlayerView
+ (Class)layerClass {
    return [AVPlayerLayer class];
}
- (AVPlayer*)player {
    return [(AVPlayerLayer *)[self layer] player];
}
- (void)setPlayer:(AVPlayer *)player {
    [(AVPlayerLayer *)[self layer] setPlayer:player];
}
@end
```

一个简单的 PlayerViewController：

```
@class PlayerView;
@interface PlayerViewController : UIViewController
 
@property (nonatomic) AVPlayer *player;
@property (nonatomic) AVPlayerItem *playerItem;
@property (nonatomic, weak) IBOutlet PlayerView *playerView;
@property (nonatomic, weak) IBOutlet UIButton *playButton;
- (IBAction)loadAssetFromFile:sender;
- (IBAction)play:sender;
- (void)syncUI;
@end
```

同步 UI 的方法：

```
- (void)syncUI {
    if ((self.player.currentItem != nil) &&
        ([self.player.currentItem status] == AVPlayerItemStatusReadyToPlay)) {
        self.playButton.enabled = YES;
    }
    else {
        self.playButton.enabled = NO;
    }
}
```

在 `viewDidLoad` 时先调用一下 `syncUI`：


```
- (void)viewDidLoad {
    [super viewDidLoad];
    [self syncUI];
}
```

创建并加载 `AVURLAsset`，在加载成功时，创建 item、初始化播放器以及添加各种监听：

```
static const NSString *ItemStatusContext;

- (IBAction)loadAssetFromFile:sender {
 
    NSURL *fileURL = [[NSBundle mainBundle] URLForResource:<#@"VideoFileName"#> withExtension:<#@"extension"#>];
 
    AVURLAsset *asset = [AVURLAsset URLAssetWithURL:fileURL options:nil];
    NSString *tracksKey = @"tracks";
 
    [asset loadValuesAsynchronouslyForKeys:@[tracksKey] completionHandler: ^{
        // The completion block goes here.
        dispatch_async(dispatch_get_main_queue(), ^{
            NSError *error;
            AVKeyValueStatus status = [asset statusOfValueForKey:tracksKey error:&error];

            if (status == AVKeyValueStatusLoaded) {
                self.playerItem = [AVPlayerItem playerItemWithAsset:asset];
                 // ensure that this is done before the playerItem is associated with the player
                [self.playerItem addObserver:self forKeyPath:@"status" options:NSKeyValueObservingOptionInitial context:&ItemStatusContext];
                [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(playerItemDidReachEnd:) name:AVPlayerItemDidPlayToEndTimeNotification object:self.playerItem];
                self.player = [AVPlayer playerWithPlayerItem:self.playerItem];
                [self.playerView setPlayer:self.player];
            } else {
                // You should deal with the error appropriately.
                NSLog(@"The asset's tracks were not loaded:\n%@", [error localizedDescription]);
            }
        });
    }];
}
```

响应 `status` 的监听通知：

```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
 
    if (context == &ItemStatusContext) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self syncUI];
        });
        return;
    }
    [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    return;
}
```

播放，以及播放完成时的处理：

```
- (IBAction)play:sender {
    [self.player play];
}


- (void)playerItemDidReachEnd:(NSNotification *)notification {
    [self.player seekToTime:kCMTimeZero];
}
```


[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-av-playback
[3]: https://developer.apple.com/library/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html