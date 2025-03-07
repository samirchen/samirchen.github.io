---
layout: post
title: 在 Mac OS 上编译 IJKPlayer
description: 介绍在 Mac OS 上编译 IJKPlayer 的流程。
category: blog
tag: Audio, Video, FFmpeg, IJKPlayer
---


在 Mac OS 上编译 IJKPlayer 的主要流程参见 [IJKPlayer 的文档][3]就可以了。

这里主要记一些遇到过的问题：

1、在最新版的 Xcode(10.0+) 上编译 ffmpeg 可能会遇到如下报错：

```
./libavutil/arm/asm.S:50:9: error: unknown directive
        .arch armv7-a
        ^
make: *** [libavcodec/arm/aacpsdsp_neon.o] Error 1
```

这个问题的原因是因为最新的 Xcode 已经弱化了对 32 位处理器的支持，解决这个问题的方法可以[参考这里][5]，有下面几种方案：

推荐使用第 1、2 种方案。第 3 种方案放弃了对 armv7 的支持，对许多应用还是不可接受的。

1）下载一个低版本的 Xcode 9.1，执行下列命令：

```
sudo xcode-select -s <pathToXcode9.1>/Contents/Developer
./compile-ffmpeg.sh clean
./compile-ffmpeg.sh all
```


2）在 `ios/tools/do-compile-ffmpeg.sh` 文件中增加如下配置：

`FFMPEG_CFG_FLAGS_ARM="$FFMPEG_CFG_FLAGS_ARM --disable-asm"`

然后再重新编译。


3）在 `compile-ffmpeg.sh` 文件中去除对 armv7 架构的支持，修改

```
FF_ALL_ARCHS_IOS8_SDK="armv7 arm64 i386 x86_64"
```

为

```
FF_ALL_ARCHS_IOS8_SDK="arm64 i386 x86_64"
```

然后再重新编译。

但是这种方案会导致后面在真机上编译 IJKMediaFramework 时报错：

```
'armv7/avconfig.h' file not found
```

只能把 `#include "armv7/avconfig.h"` 的代码注释掉。



2、如果要支持 HTTPS，需要按如下步骤来：

```
# 获取 openssl 并初始化
./init-ios-openssl.sh

# 在 module.sh 文件中添加一行配置以启用 openssl 组件
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-openssl"


./compile-ffmpeg.sh clean

# 编译 openssl
./compile-openssl.sh all

# 编译 ffmpeg
./compile-ffmpeg.sh all
```

然后，在 IJKMediaDemo 工程中要把 IJKMediaFramework 库替换为 IJKMediaFrameworkWithSSL。



其他参考：

- [ijkplayer 的编译、打包 framework 和 https 支持][4]



更多的音视频技术文章可以去看：[关键帧Keyframe：https://gjzkeyframe.github.io/](https://gjzkeyframe.github.io/)



[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/complie-ijkplayer-on-mac-os
[3]: https://github.com/bilibili/ijkplayer
[4]: https://blog.wskfz.com/index.php/archives/59/
[5]: https://github.com/Bilibili/ijkplayer/issues/4150

