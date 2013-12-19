---
layout: post
title: Objective-C中的property
description: 介绍Objective-C中的property相关知识。
category: blog
tag: iOS, Objective-C, property
---

本文中的概念都是基于 iOS 7 的，在以前版本中是否适用并未做详细考证。

###private & public
在 [Objective-C][3] 中，`.h` 文件中都是 public API 的声明，`.m` 文件中是 private API 的声明和所有的代码实现。比如：

`Card.h` 文件：

	#import <Foundation/Foundation.h>
	@interface Card : NSObject
	@property (nonatomic, strong) NSString* publicName; // public property.
	-(void) publicfunc; // public method.
	@end

`Card.m` 文件：

	#import "Card.h"
	@interface Card()
	@property (nonatomic, strong) NSString* privateName; // private property.
	@end

	@implementation Card
	-(void) publicFunc { // public method implementation.
		// ...
	}

	-(void) privateFunc { // private method implementation.
		// ...
	}
	@end


###strong & weak
`strong` 关键字和 `weak` 关键字用于对指针属性（@property）的修饰。

当堆上的对象有一个或多个 strong 指针指向它时，它会安好。当一个指向它的 strong 指针也没有时（比如设置最后一个指向它的 strong 指针为 nil），这个对象立即被释放；如果此时还有一个或多个 weak 的指针指向这个对象，那么这些 weak 的指针将被设为 nil。比如：

	// 下面使用的 __strong 和 __weak 关键字一般用于修饰局部变量，其对应的内存管理机制是一致的：
	__strong A* stronga = [[A alloc] init]; // 创建一个对象
	__weak A* weaka = stronga; // 用一个 weak 的指针指向这个对象
	stronga = nil; // strong 指针被设为 nil，对象立即被释放，此时 weaka 也被设置为了 nil

像 NSInteger、BOOL 这样的基本类型则不需要 strong 或 weak 的修饰，它们不是指针。比如：

	@property (nonatomic) BOOL isOK; // 这样就可以了

`nonatomic` 关键字用于对属性的修饰，当调用这个属性的 setter 或 getter 的时候不是线程安全的，这样开销也会小一些。

###@synthesize
`@synthesize` 的含义是在没有进行重载的情况下，让编译器根据读写属性自动为类的属性生成 getter 和 setter 方法。我们通常会这样写：

`.h` 文件中：

	@property (nonatomic, strong) NSString* contents;

`.m` 文件中：

	@synthesize contents = _contents; 

表示 `_contents` 是 contents 这个属性的名字，在 `_contents` 里来为属性分配空间来存储它。这个 `_contents` 我们通常可以在这个属性的 getter 和 setter 方法里用它。比如：

	// getter
	-(NSString*) contents {
	     return _contents;
	}
	// setter
	-(void) setContents:(NSString*)contents {
	     _contents = contents;
	}

如果我们不自己写 getter、setter，`@synthesize` 会让编译器自动帮我们生成，类似上面的代码。

在 getter 和 setter 外的其他地方，我们不要贸然使用 `_contents` 而要使用 `self.contents`，因为 `_contents` 是不会调用这个属性的 getter 或 setter 的，而有时候我们会在 getter 或 setter 里添加额外的代码，希望属性在被获取或被设置的时候调用这些代码，那如果直接用 `_contents` 就达不到这个目的了。通过 `self.contents` 的方式会调用这个 property 对应的 getter 方法；通过 `self.contents = ...;` 的方式会调用这个 property 对应的 setter 方法。当然我们也可以用 `[ ]` 的方式来调用 getter 和 setter，习惯问题。

有时候，想太多，我们想在代码中不用 `self.contents` 也不用 `_contents`，而是用 `contents`，这是编译器会报错，提醒你：你是想用 `_contents` 吧？现在属性 `@property contents` 的名字 `_contens`。

在 iOS 7 中可以不用写 `@synthesize` 了，编译器可以默认帮助自动生成 `@synthesize contents = _contents;` 以及对应的 getter 和 setter。

要注意一点，[Objective-C][3] 对象的所有的属性（即实例变量instance variables）都初始化为 0 或 nil。@synthesize 不会为属性分配空间。在Objective-C 中向为 nil 发送消息不会造成程序崩溃，但是什么也不会执行，如果有返回值的话，返回 0 或 nil。


###@property的初始化
当我们有一个属性

	@property (strong, nonatomic) NSMutableArray *cards;

它默认的 getter 是这样的：

	-(NSMutableArray*) cards {
	     return _cards;
	}

这样的话，因为 [Objective-C][3] 对象的所有的实例变量（instance variables）都初始化为 0 或 nil，所以当我们要用 cards 这个属性的时候得到的是 nil。我们通常不希望这样，那最好的初始化属性的地方是在它的 getter 方法中，我们可以这样做：

	-(NSMutableArray*) cards {
	     if (!_cars) {
	          _cards = [NSMutableArray alloc] init];
	     }
	     return _cards;
	}

我们通常都是在 getter 中初始化属性，而不是在别的地方搞个 initializer 之类的方法来做。这种方式叫做 [Lazy Instantiation][4]。这是 Objective-C 中常用的模式。

###IBAction & IBOutlet
从界面上的一个控件 Control+drag 拖一个方法到代码的实现区，会得到类似：

	-(IBAction) touchCardButton:(UIButton*)sender;

这里的 `IBAction` 其实返回的是 void，类似 `typedef IBAction void`。IBAction 是为了让 Xcode 去区别方法是不是 target action，这样我们可以做到鼠标在上面的时候界面会高亮对应的控件之类的。但是编译器会忽略它，对编译器来说它就是 void。当你要改这个方法名的时候，有点麻烦的是你需要对着对应的控件点右键，然后叉掉方法绑定，然后重新连接。

当我们从界面上按住Option拖一个控件到代码的属性区，会得到类似：

	@property (weak, nonatomic) IBOutlet UILabel* flipsLabel;

的UI属性。这里一定是一个 weak 的属性，因为这个属性是被 View 所 strong 持有的，对 Controller 来说只应该 weak 地指向它。这里的 IBOutlet 跟 IBAction 的意思差不多，主要是方便 Xcode用它，编译器会忽略它。

###Data & UI
我们通常在属性的 setter 中保证属性数据和对应的 UI 的一致性。比如：

	-(void) setFlipCount:(int)flipCount {
	     _flipCount = flipCount;
	     self.flipsLabel.text = [NSString stringWithFormat:@"Flips: %d", self.flipCount];
	}



[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://samirchen/property-in-objectivec
[3]: http://zh.wikipedia.org/zh-cn/Objective-C
[4]: http://en.wikipedia.org/wiki/Lazy_initialization