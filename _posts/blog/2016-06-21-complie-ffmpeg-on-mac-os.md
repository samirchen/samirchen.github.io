---
layout: post
title: 在 Mac OS 上编译 FFmpeg
description: 介绍在 Mac OS 上编译 FFmpeg 的流程。
category: blog
tag: Audio, Video, FFmpeg
---

## 安装 Xcode 和 Command Line Tools

从 App Store 上安装 Xcode，并确保在 Xcode 的 `Preferences -> Downloads -> Components` 下安装好 Command Line Tools。

当然你也可以从 [https://developer.apple.com/][5] 下载 Xcode 和 Command Line Tools。


## 安装 brew

[Homebrew][6] 是 Mac 上的一个很好用的包管理工具，安装方法即允许下列命令：

```
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```


## 使用 brew 安装依赖库

```
$ brew install automake fdk-aac git libtool libvorbis libvpx opus sdl shtool yasm texi2html theora wget x264 xvid lame libass
```

在安装这些库时，如果发生错误，可以重试一下，有时候可能是由于网络原因导致下载未完成而引起安装失败。你可以这样来单独安装一个库：

```
// Install x264 with brew.
$ brew install x264
```

如果有的库始终安装不成功，那么你可以尝试先升级更新下 brew：

```
brew update
```

悲剧的是，有时候执行 `brew update` 后，brew 可能都报错了，原因大多是本地的 brew 仓库（通常在 /usr/local/ 目录下）发生了冲突，这时候需要执行下 git 命令处理下冲突再更新 brew，命令如下：

```
$ cd $(brew --prefix)
$ git reset --hard HEAD
$ brew update
```

如果你还遇到其他问题，就先 Google 一下来解决吧。







## 编译 FFmpeg

接着就是用下列命令下载 FFmpeg 源码和编译它：

```
// 下载 FFmpeg 源码：
$ git clone http://source.ffmpeg.org/git/ffmpeg.git ffmpeg
// 编译：
$ cd ffmpeg
$ ./configure  --prefix=/usr/local --enable-gpl --enable-nonfree --enable-libass \
--enable-libfdk-aac --enable-libfreetype --enable-libmp3lame --enable-libopus \
--enable-libtheora --enable-libvorbis --enable-libvpx --enable-libx264 --enable-libxvid --extra-ldflags=-L/usr/local/lib
$ make && make install
```

当你 config 的时候有时候会报错找不到一些库，这时候你可以添加 `--extra-ldflags=-L/usr/local/lib` 试试。


## 测试一下

编译完成不报错的话，接下来你就可以试试拿一个视频来播着试试了，在 FFmpeg 目录下执行下面的命令让 FFmpeg 播放一个视频：

```
$ ffplay http://devimages.apple.com.edgekey.net/streaming/examples/bipbop_16x9/gear5/prog_index.m3u8
```


[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/complie-ffmpeg-on-mac-os
[3]: https://trac.ffmpeg.org/wiki/CompilationGuide/MacOSX
[4]: http://stackoverflow.com/questions/26461966/osx-10-10-curl-post-to-https-url-gives-sslread-error/26538127
[5]: https://developer.apple.com/
[6]: http://brew.sh/
