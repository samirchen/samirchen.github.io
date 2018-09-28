---
layout: post
title: iOS 应用的启动过程
description: 分析一下 iOS 应用的启动过程。
category: blog
tag: iOS
---


## 应用入口

通常，说到一个 App 的入口，我们会想到 `main()` 函数，但其实在 `main()` 调用之前还有一段过程，我们称之为 `pre-main`。


在 Xcode 新建一个项目，是通过 main() 来引导启动了一个 UIApplicationMain 方法来对 AppDelegate 进行回调。当断点在 main 函数里时，我们发现调用栈底多了一个 start 方法。这时的堆栈是：

```
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  * frame #0: 0x00000001025d27ec MyApp`main(argc=1, argv=0x000000016d8339f8) at main.m:13
    frame #1: 0x000000018212dfc0 libdyld.dylib`start + 4
```

进入 start 栈帧可以看到如下代码：

```
libdyld.dylib`start:
    0x18212dfbc <+0>: nop    
    0x18212dfc0 <+4>: bl     0x182176244               ; exit
    0x18212dfc4 <+8>: brk    #0x3
```


通过注释可以猜到这是进入程序退出步骤的代码，也就是 main 函数执行后最终的去处。但是能发现 start() 方法是 libdyld.dylib 动态库中的。

这是在 main() 中发现的全部信息，我们再换一种方式去进入这个 pre-main：`+load` 函数。

+load() 是 NSObject 的类方法，我们都知道很多早于 main() 执行的代码都可以在继承类的 +load() 里完成。通过在 AppDelegate 类添加一个 +load() 方法，断点查看调用栈：


```
(lldb) thread backtrace
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
  * frame #0: 0x00000001025d2884 MyApp`+[AppDelegate load](self=AppDelegate, _cmd="load") at AppDelegate.m:19
    frame #1: 0x00000001819929f0 libobjc.A.dylib`call_load_methods + 184
    frame #2: 0x0000000181993b58 libobjc.A.dylib`load_images + 76
    frame #3: 0x00000001027060c8 dyld`dyld::notifySingle(dyld_image_states, ImageLoader const*, ImageLoader::InitializerTimingList*) + 384
    frame #4: 0x000000010271612c dyld`ImageLoader::recursiveInitialization(ImageLoader::LinkContext const&, unsigned int, char const*, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 440
    frame #5: 0x00000001027151cc dyld`ImageLoader::processInitializers(ImageLoader::LinkContext const&, unsigned int, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 136
    frame #6: 0x0000000102715288 dyld`ImageLoader::runInitializers(ImageLoader::LinkContext const&, ImageLoader::InitializerTimingList&) + 84
    frame #7: 0x0000000102706498 dyld`dyld::initializeMainExecutable() + 220
    frame #8: 0x000000010270b0fc dyld`dyld::_main(macho_header const*, unsigned long, int, char const**, char const**, char const**, unsigned long*) + 6176
    frame #9: 0x000000010270521c dyld`_dyld_start + 68
```


可以发现此时还未执行到 main()，而且看起来调用函数也不少。不过暂且忽略具体内容，直接进入最底部的 `_dyld_start()` 栈帧，在最底下的 br 指令加一个断点，并执行到此处，显示如下汇编指令：


```
dyld`_dyld_start:
    0x1027051d8 <+0>:   mov    x28, sp
    0x1027051dc <+4>:   and    sp, x28, #0xfffffffffffffff0
    0x1027051e0 <+8>:   mov    x0, #0x0
...
    0x102705218 <+64>:  bl     0x102705260               ; dyldbootstrap::start(macho_header const*, int, char const**, long, macho_header const*, unsigned long*)
    0x10270521c <+68>:  mov    x16, x0
...
->  0x10270525c <+132>: br     x16
```

上述代码大致经历了以下步骤：


- 1、移动栈底指针以及局部变量初始化等操作。
- 2、调用 `dyldbootstrap::start()` 方法。
- 3、将 `dyldbootstrap::start()` 的返回值会放入 x0 寄存器，随后赋给 x16 寄存器。
- 4、跳转 x16 寄存器所存储的地址。

那 x16 寄存器存的地址是什么？我们通过 lldb 查看：

```
(lldb) register read x16
     x16 = 0x00000001025d27d4  MyApp`main at main.m:12
```

通过注释说明，这正是指向 main() 函数的地址，结合开头我们获取的 main 的调用栈地址 0x00000001026ea7ec，br 后就是进入了 main 函数的主体代码，相差的几个字节是压栈时移动栈底指针的汇编指令。


到这里就知道了 main() 的源头，但又产生更多的疑问，`dyldbootstrap::start()` 里面做了什么？是如果得到并返回 main() 函数地址的？调用栈里出现频繁的的 dyld/libdyld 是什么？而这个 `_dyld_start` 又看起来比 main() 更像入口一点。

另外，仔细观察上述汇编指令的地址，有些是 0x102 开头的连续地址，而有些是开头 0x181 的地址，就运行情况来说 0x102 开头对应的函数地址每次运行都会改变，0x181 的那些地址就是固定的，这里有什么原因和差异。



[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-app-launching