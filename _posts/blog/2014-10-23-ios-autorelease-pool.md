---
layout: post
title: iOS中的AutoreleasePool
description: 介绍iOS中的AutoreleasePool相关。
category: blog
tag: iOS, autorelease pool, memory management
---

文章的主要内容源自苹果的官方文档：[Using Autorelease Pool Blocks][3]


##关于Autorelease Pool
[Autorelease Pool][3] 是iOS内存管理机制中很重要的一个部分。 Autorelease Pool字面上看起来是「自动释放池」的意思，在这个池子里去管理对象的内存。

Autorelease Pool的一个使用场景是在需要延迟释放某些对象的情况时，可以把他们先放到对应的Autorelease Pool中，等Autorelease Pool生命周期结束时再一起释放。这些对象会被发送autorelease消息。在非ARC时代，我们显示地发送autorelease消息，在ARC时代，系统会帮我们做这些事情。[点这里了解更多][4]

我们使用Autorelease Pool时候的语法大致如下：

	@autoreleasepool {
	    // Code that creates autoreleased objects.
	}

而且可以嵌套：

	@autoreleasepool {
	    // . . .
	    @autoreleasepool {
	        // . . .
	    }
	    . . .
	}

Cocoa中总是会期望代码在一个Autorelease Pool Block中运行，否则那些被autorelease的对象就不能被释放了，这就内存泄露了。在AppKit和UIKit框架中，当处理一个事件循环的一次迭代的时候（比如一次鼠标点击事件或一次tap事件），会自动在一个autorelease pool block里完成，因此通常你不需要自己去创建一个autorelease pool block，甚至你都很少看到这种代码。

但是，你仍然会在下列几种情况下用到你自己的autorelease pool block：

- 你编写是命令行工具的代码，而不是基于UI框架的代码。
- 你需要写一个循环，里面会创建很多临时的对象。
	- 这时候你可以在循环内部的代码块里使用一个autorelease pool block，这样这些对象就能在一次迭代完成后被释放掉。这种方式可以降低内存最大占用。
- 当你大量使用辅助线程。
	- 你需要在线程的任务代码中创建自己的autorelease pool block。


##使用Autorelease Pool Block来降低内存占用峰值

上例子：

	NSArray *urls = <# An array of file URLs #>;
	for (NSURL *url in urls) {
	 
	    @autoreleasepool {
	        NSError *error;
	        NSString *fileContents = [NSString stringWithContentsOfURL:url
	                                         encoding:NSUTF8StringEncoding error:&error];
	        /* Process the string, creating and autoreleasing more objects. */
	    }
	}

上面的例子中，for循环的每次迭代处理一个文件，我们把这个处理过程放在一个autorelease pool block中，这样任何在里面创建的autorelease的对象都会在这次迭代结束后被释放，这样就不会占用很多内存了。

在autorelease pool block执行完后，所有曾在里面autorelease的对象都可以被认为是释放了的，所以就别再给它们发消息了。**如果你确实要穿越autorelease pool block用到临时对象**，那可以在autorelease pool block里retain它，之后再autorelease它，看下面例子：

	– (id)findMatchingObject:(id)anObject {
	 
	    id match;
	    while (match == nil) {
	        @autoreleasepool {
	 
	            /* Do a search that creates a lot of temporary objects. */
	            match = [self expensiveSearchForObject:anObject];
	 
	            if (match != nil) {
	                [match retain]; /* Keep match around. */
	            }
	        }
	    }
	 
	    return [match autorelease];   /* Let match go and return it. */
	}


##Autorelease Pool Block和线程
在Cocoa中，每个线程去维护它自己的autorelease pool block的栈，所以当我们自己要写一些线程代码，我们可能就要自己去创建自己的autorelease pool block了，尤其当你的线程是长时间工作、可能生产出大量的autorelease的对象的时候。

如果你分发的线程不会产生对Cocoa的调用，那么你也许就不需要使用autorelease pool block了。













[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://samirchen.com/ios-autorelease-pool/
[3]: https://developer.apple.com/library/ios/documentation/cocoa/conceptual/memorymgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-CJBFBEDI
[4]: http://www.galloway.me.uk/2012/02/a-look-under-arcs-hood-episode-3/
