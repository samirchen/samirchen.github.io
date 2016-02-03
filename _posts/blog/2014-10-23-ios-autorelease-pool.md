---
layout: post
title: iOS中的AutoreleasePool
description: 介绍iOS中的AutoreleasePool相关。
category: blog
tag: iOS, autorelease pool, memory management
---

文章的主要内容源自苹果的官方文档：[Using Autorelease Pool Blocks][3]


## 关于Autorelease Pool

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


## 使用Autorelease Pool Block来降低内存占用峰值


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


## Autorelease Pool Block和线程

在Cocoa中，每个线程去维护它自己的autorelease pool block的栈，所以当我们自己要写一些线程代码，我们可能就要自己去创建自己的autorelease pool block了，尤其当你的线程是长时间工作、可能生产出大量的autorelease的对象的时候。

如果你分发的线程不会产生对Cocoa的调用，那么你也许就不需要使用autorelease pool block了。



## autorelease的实现


既然说到 Autorelease Pool，这里就多说一下在 MRC 时代 autorelease 的实现机制，了解这个机制对于我们理解 Objective-C 的内存管理是很有帮助的。

在讨论实现之前，我们首先来看个问题，我们都知道 autorelease 机制的一个典型的应用场景就是参数传递返回值。那为什么方法返回值的时候需要用到 autorelease 特性呢？这是因为你在方法中创建了一个对象，在返回之前你总不能把它释放了吧，这样你返回的就是 nil 了，而返回后，方法都调用结束了。参数值 return 出去后，外面接收它的要使用它就要强引用它，给它 retainCount +1，不用了就清理，retainCount -1，这是方法外面的事，但对于你这个方法来说你创建了它却没清理它，这是不负责任地搞内存泄露啊！所以得有一套机制保证创建这个对象的方法能在方法调用结束后清理掉这个对象，这就是 autorelease 机制。

在 Objective-C 中编程人员可以通过 autorelease 机制设定变量的作用域，这其中就离不开 Autorelease Pool。结合 Autorelease Pool 来使用 autorelease 的方法大致如下：

	NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];  
	id obj = [[NSObject alloc] init];  
	[obj autorelease];  
	// Do something...
	[pool drain]; 

上面的代码最后一行执行的时候就会自动调用 `[obj release];`。

这个过程中调用 `[obj autorelease]` 究竟发生了什么呢？我们可以查看 GNUstep 的源代码来看看：

	// GNUstep/modules/core/base/Source/NSObject.m autorelease
	- (id)autorelease {  
		[NSAutoreleasePool addObject:self];  
	}

autorelease 实例方法的本质就是调用 NSAutoreleasePool 对象的 addObject 类方法。下面来看看这个类方法的实现，由于源码比较复杂，下面对代码进行了简化，大致如下：

	// GNUstep/modules/core/base/Source/NSAutoreleasePool.m addObject
	+ (void)addObject:(id)anObj {  
		NSAutoreleasePool *pool = 取得正在使用的 NSAutoreleasePool 对象;  
		if (pool != nil) {  
			[pool addObject:anObj];  
		} else {  
			NSLog(@"NSAutoreleasePool 对象非存在状态下调用 autorelease");  
		}  
	}

什么叫`正在使用的 NSAutoreleasePool 的对象`呢？比如下面：

	NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];  
	id obj = [[NSObject alloc] init];  
	[obj autorelease]; 

被赋值的 pool 变量就是。

而这个例子里面：

	NSAutoreleasePool *pool0 = [[NSAutoreleasePool alloc] init];  
		NSAutoreleasePool *pool1 = [[NSAutoreleasePool alloc] init];  
			NSAutoreleasePool *pool2 = [[NSAutoreleasePool alloc] init];  
				id obj = [[NSObject alloc] init];  
				[obj autorelease];  
			[pool2 drain];  
		[pool1 drain];  
	[pool0 drain];

则是最内侧的 pools。

下面看看实例方法的实现：

	// GNUstep/modules/core/base/Source/NSAutoreleasePool.m addObject
	- (void)addObject:(id)anObj   {  
		[array addObject:anObj];  
	} 

实际的 GNUstep 实现使用的是链表结构，这跟在 NSMutableArray 中添加对象是一样的。如果调用 NSObject 类的 autorelease 实例方法，该对象将被追加到正在使用的 NSAutoreleasePool 对象中的数组里。

再看看看 `[pool drain];` 的实现。

	//GNUstep/modules/core/base/Source/NSAutoreleasePool.m drain
	- (void)drain {  
		[self dealloc];  
	}  
	- (void)dealloc {  
		[self emptyPool];  
		[array release];  
	}  
	- (void)emptyPool {  
		for (id obj in array) {  
			[obj release];  
		}  
	} 

调用了好几个方法，最终是对于 Autorelease Pool 的数组里的对象都调用了 release 方法。



## NSThread、NSRunLoop 和 NSAutoreleasePool

根据苹果官方文档中对 NSRunLoop 的描述，我们可以知道每一个线程，包括主线程，都会拥有一个专属的 NSRunLoop 对象，并且会在有需要的时候自动创建。

在主线程的 [NSRunLoop][5] 对象（在系统级别的其他线程中应该也是如此，比如通过 dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0) 获取到的线程）的每个 event loop 开始前，系统会自动创建一个 Autorelease Pool ，并在 event loop 结束时 drain 。我们上面提到的场景 1 中创建的 autoreleased 对象就是被系统添加到了这个自动创建的 Autorelease Pool 中，并在这个 Autorelease Pool 被 drain 时得到释放。

另外，[NSAutoreleasePool][6] 中还提到，每一个线程都会维护自己的 Autorelease Pool 堆栈。换句话说 Autorelease Pool 是与线程紧密相关的，每一个 Autorelease Pool 只对应一个线程。

弄清楚 NSThread、NSRunLoop 和 NSAutoreleasePool 三者之间的关系可以帮助我们从整体上了解 Objective-C 的内存管理机制



[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://samirchen.com/ios-autorelease-pool/
[3]: https://developer.apple.com/library/ios/documentation/cocoa/conceptual/memorymgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-CJBFBEDI
[4]: http://www.galloway.me.uk/2012/02/a-look-under-arcs-hood-episode-3/
[5]: https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/doc/constant_group/Run_Loop_Modes
[6]: https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/doc/constant_group/Run_Loop_Modes