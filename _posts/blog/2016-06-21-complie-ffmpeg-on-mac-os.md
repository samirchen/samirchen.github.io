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
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```


## 使用 brew 安装依赖库

```
brew install automake fdk-aac git libtool libvorbis libvpx opus sdl shtool yasm texi2html theora wget x264 xvid lame libass
```

在安装这些库时，如果发生错误，可以重试一下，有时候可能是由于网络原因导致下载未完成而引起安装失败。你可以这样来单独安装一个库：

```
// Install x264 with brew.
brew install x264
```

如果有的库始终安装不成功，那么你可以尝试先升级更新下 brew：

```
brew update
```

悲剧的是，有时候执行 `brew update` 后，brew 可能都报错了，原因大多是本地的 brew 仓库（通常在 /usr/local/ 目录下）发生了冲突，这时候需要执行下 git 命令处理下冲突再更新 brew，命令如下：

```
cd $(brew --prefix)
git reset --hard HEAD
brew update
```

如果你还遇到其他问题，就先 Google 一下来解决吧。







## 编译 FFmpeg







[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/complie-ffmpeg-on-mac-os
[3]: https://trac.ffmpeg.org/wiki/CompilationGuide/MacOSX
[4]: http://stackoverflow.com/questions/26461966/osx-10-10-curl-post-to-https-url-gives-sslread-error/26538127
[5]: https://developer.apple.com/
[6]: http://brew.sh/
