---
layout: post
title: 点播中的卡顿优化
description: 介绍在点播中进行卡顿优化的一些思路。
category: blog
tag: Audio, Video, MP4, Stall
---

视频播放卡顿是音视频业务里一个常见的问题，引起视频播放卡顿的主要原因通常包括：

- 网络带宽速度不够，造成音视频数据无法及时完成下载。
- 设备解码性能不足，造成音视频数据无法及时完成解码或渲染。

基于这些原因，我们可以考虑下面的方案来对播放过程中的卡顿进行优化。


## 使用 HTTPDNS 优化网络连接

在现在的网络视频播放场景中，对视频资源的访问通常都要经过 CDN 网络进行内容分发和调度。如果 CDN 线路质量不好，那么久很可能造成播放时音视频数据下载的延迟，从而引起播放的卡顿。

这时候我们可以使用与 CDN 网络配套的 HTTPDNS 服务。HTTPDNS 使用 HTTP 协议进行域名解析，代替现有基于 UDP 的 DNS 协议，域名解析请求直接发送到相应的 HTTPDNS 服务器，从而提供更优的线路选择。

以 iOS 上的 AVPlayer 为例，当使用 HTTPDNS 时，可以用视频资源 URL 对应的 Host 向 HTTPDNS 请求节点 IP，然后用节点 IP 替换 URL 中的 Host 部分，再在 HTTP Header 里设置原 Host。这样即可通过 IP 直连的方式访问 HTTPDNS 返回的较优节点。

示例代码大致如下：


```
// 假设原视频 URL 是：http://wwww.example.com/abc.mp4
// 假设从 HTTPDNS 服务获取的 wwww.example.com 这个 Host 对于的 IP 是：192.168.1.1
// 那么处理后的 URL 是：http://192.168.1.1/abc.mp4

NSMutableDictionary *headers = [NSMutableDictionary dictionary];
[headers setObject:@"wwww.example.com" forKey:@"Host"];
NSURL *videoURL = [NSURL URLWithString:@"http://192.168.1.1/abc.mp4"];
AVAsset *asset = [AVURLAsset URLAssetWithURL:videoURL options:@{@"AVURLAssetHTTPHeaderFieldsKey": headers}];
AVPlayerItem *playerItem = [AVPlayerItem playerItemWithAsset:asset];
AVPlayer *player = [AVPlayer playerWithPlayerItem:playerItem];
```

这种方案在使用 HTTPS 时，是会失败的。因为 HTTPS 在证书验证的过程，会出现 domain 不匹配导致SSL/TLS握手不成功。这时候的方案参考[HTTPS（含SNI）业务场景“IP直连”方案说明][3]和[iOS HTTPS SNI 业务场景“IP直连”方案说明][4]。

## 优化缓冲策略

在点播场景下，为了减少播放过程中的卡顿，可以在缓冲一定的数据后再解码播放。但是这样，就会影响视频的首屏播放速度。

增大播放器的缓冲区，使得每次下载时能够加载足够的数据再进行播放，能够降低播放过程中卡顿的频次，但是这样也会延长首屏播放速度以及每次卡顿后恢复播放的速度。所以对于缓冲区的大小的设置，需要考虑卡顿和快速开播两个因素，尽量取得平衡。

在 iOS 平台上，使用系统的 AVPlayer 时，属性 `automaticallyWaitsToMinimizeStalling` 就是控制播放器缓冲策略的。当该值为 YES 时，AVPlayer 会努力尝试延迟开始播放，加载足够的数据来保证整个播放过程中尽量卡顿最少。这个接口在 iOS 10 及以上版本才开放，在 iOS 10 之前的版本，在播放 HLS 这种流媒体视频时，效果如同 `automaticallyWaitsToMinimizeStalling` 为 YES，播放基于文件的视频资源，包括通过网络传输的网络视频文件，则效果如同 `automaticallyWaitsToMinimizeStalling` 为 NO。


## 支持硬解

对于清晰度较高的视频，对于解码的性能消耗也会较大，如果设备的性能不足，则可能会造成解码速度赶不上播放速度，从而造成卡顿。在实际情况中，可以尽量选择使用硬解，通过 GPU 来进行解码。


## 码流切换

720P、1080P 等清晰度较高的码流，对于网速的要求以及设备性能的要求都会相对较高，在发生卡顿的情况下，可以考虑将播放的视频切换到较低清晰度的码流，从而优化网络加载速度，降低对设备性能的消耗，优化视频播放的卡顿。







[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/video-playback-stall
[3]: https://help.aliyun.com/document_detail/30143.html
[4]: https://help.aliyun.com/knowledge_detail/60147.html
