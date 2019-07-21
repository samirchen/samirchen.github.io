---
layout: post
title: 在 Mac OS 上搭建 nginx+rtmp 服务器
description: 介绍在 Mac OS 上在 Mac OS 上搭建 nginx+rtmp 服务器来做直播推流服务。
category: blog
tag: Audio, Video, nginx, rtmp, live
---


## 搭建 nginx 流程

1、安装 HomeBrew。

```
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

2、安装 nginx。

克隆 GitHub 的项目 (https://github.com/denji/homebrew-nginx)

```
brew tap denji/nginx
```

安装 xcode 工具：

```
xcode-select --install
```

安装 nginx + rtmp：

```
brew install nginx-full --with-rtmp-module
```

nginx 安装所在位置和配置文件所在的位置：

```
/usr/local/opt/nginx-full/bin/nginx
/usr/local/etc/nginx/nginx.conf
``` 

3、运行 nginx。

启动 nginx，执行命令:

```
nginx
```

在浏览器地址栏输入：[http://localhost:8080](http://localhost:8080)

应该可以看到 `Welcome to nginx!` 的页面。

其他使用 nginx 相关的问题：

1）端口号被占用的问题：

```
// nginx 的报错表示 8080 端口号被占用：
nginx: [emerg] bind() to 0.0.0.0:8080 failed (48: Address already in use)

// kill 掉占用 8080 端口号的进程：
// 查看占用 8080 端口号的进程：
lsof -i tcp:8080
// 干掉该进程：
kill <PID>
```

2）重新加载和重启

```
// 重新加载配置文件：
nginx -s reload

// 停止：
nginx -s stop

// 有序退出：
nginx -s quit
```

4、配置 RTMP。

打卡 nginx 的配置文件：

```
vi /usr/local/etc/nginx/nginx.conf
```

在

```

http {
	...
}
```

节点后，增加 rtmp 配置：

```
rtmp {
    server {
        listen 1935;

        # 直播流配置
        application rtmplive {
            live on;

            # 为 rtmp 引擎设置最大连接数，默认为 off。
            max_connections 1024;
        }
        application hlslive {
            live on;
            hls on;
            hls_path /usr/local/var/www/hls;
            hls_fragment 1s;
        }
    }
}
```

编辑完成之后，执行一下重新加载配置文件命令：

```
nginx -s reload
```

重启 nginx：


## 使用 FFmpeg 来推流

在 Mac OS 上按照 FFmpeg 的流程参见：[在 Mac OS 上编译 FFmpeg][4]

使用 FFmpeg 推流：

```
ffmpeg -re -i /Users/xxx/Downloads/test.mp4 -vcodec libx264 -acodec aac -strict -2 -f flv rtmp://localhost:1935/rtmplive/room
```

然后在播放器上，播放该推流即可：

```
rtmp://localhost:1935/rtmplive/room
```


## 参考

- [Mac 搭建 nginx+rtmp 服务器][3]


[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/install-nginx-rtmp-on-mac-os
[3]: https://blog.csdn.net/zcvbnh/article/details/79495285
[4]: http://www.samirchen.com/complie-ffmpeg-on-mac-os