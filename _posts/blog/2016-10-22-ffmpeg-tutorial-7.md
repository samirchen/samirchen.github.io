---
layout: post
title: FFmpeg 入门(7)：Seeking
description: FFmpeg 入门教程系列。
category: blog
tag: FFmpeg, tutorial
---





以上便是我们这节教程的全部内容，其中的完整代码你可以从这里获得：[https://github.com/samirchen/TestFFmpeg][6]

## 编译执行

你可以使用下面的命令编译它：

```
$ gcc -o tutorial07 tutorial07.c -lavutil -lavformat -lavcodec -lswscale -lz -lm `sdl-config --cflags --libs`
```

找一个视频文件，你可以这样执行一下试试：

```
$ tutorial07 myvideofile.mp4
```





[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ffmpeg-tutorial-7
[3]: http://dranger.com/ffmpeg/tutorial07.html
[6]: https://github.com/samirchen/TestFFmpeg