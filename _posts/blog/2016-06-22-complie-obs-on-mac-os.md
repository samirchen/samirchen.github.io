---
layout: post
title: 在 Mac OS 上编译 OBS
description: 介绍在 Mac OS 上编译 OBS 的流程。
category: blog
tag: Audio, Video, OBS
---


## 安装环境

第一步，做准备工作，安装编译 OBS 所需要的环境，流程如下：

```
// 给当前用户添加 /usr/local 文件夹的写权限，否则后面可能在安装其他环境时可能因为权限问题可遇到错误：
sudo chown -R <your-user-name> /usr/local
sudo chmod -R u+w /usr/local

// 用 brew 安装 FFmpeg。如果本地源码编译过 FFmpeg 并 make install 了，先去 FFmpeg 的源码位置 make uninstall 一下再用 brew 安装 FFmpeg：
brew install ffmpeg

// 安装 X264：
brew install x264

// 安装 Qt5 以及设置环境变量：
brew install qt5
brew linkapps qt5
export CMAKE_PREFIX_PATH=/usr/local/opt/qt5

// 安装 CMake：
brew install cmake
brew link cmake
```

## 下载和编译 OBS


从 [https://github.com/jp9000/obs-studio](https://github.com/jp9000/obs-studio) 下载 OBS 代码：

```
git clone --recursive https://github.com/jp9000/obs-studio.git
```

编译 OBS 源码：


```
// 进入 obs-studio 源码根目录，创建 build 文件夹：
cd obs-studio
mkdir build
cd build
cmake ..
make
```

## 运行 OBS

```
cd rundir/RelWithDebInfo/bin/
./obs
```

如果你想要编译 OBS 安装包，你可以在编译的时候使用 `make package` 命令即可编译出包含 OBS App Bundle 的 `.dmg` 安装包。

## 其他

你可以 [https://github.com/jp9000/obs-studio/wiki/Install-Instructions#mac-osx][3] 了解更多。

你还可以从 [https://obsproject.com/download][4] 直接下载 OBS 安装包来安装 OBS。



更多的音视频技术文章可以去看：[关键帧Keyframe：https://gjzkeyframe.github.io/](https://gjzkeyframe.github.io/)


[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/complie-obs-on-mac-os
[3]: https://github.com/jp9000/obs-studio/wiki/Install-Instructions#mac-osx
[4]: https://obsproject.com/download
