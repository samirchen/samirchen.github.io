---
layout: post
title: Block
description: 介绍Objective-C中的block。
category: blog
tag: iOS, Objective-C, block
---

##Block是什么
Block是Apple在C语言的基础上添加的扩展功能，由于Ojective-C、C++都是源自于C，所以这三种语言都能够使用。对于Block的功能，可以用一句话概括：**能持有作用域变量的匿名函数。**

`匿名函数`就是没有名称的函数，C语言的标准不允许存在这样的函数，而通过Block，源代码中就可以使用匿名函数了。

`能持有作用域变量`就是指Block能够获得其所在作用域的变量。其中其所在作用域的变量就包括：局部变量（自动变量）、函数的参数、静态局部变量、静态全局变量、全局变量。


##Block语法
Block能够让我们创建明显的代码片段，并且可以像参数那样传递给方法或函数。在Objective-C中，Block就像对象一样，能够被添加到集合中（比如：NSArray、NSDictionary）。Block能够获得其所在作用域的变量，就如同其他语言里`闭包（Closure）`或者`lambda计算`的概念。

###最简单的Block
一个简单的 Block，不返回任何值，不接受任何参数：

	// 一个简单的block：
	^{
    	NSLog(@"This is a block");
    }
    
	// 完整的写法：
	//// 声明
	void (^simpleBlock)(void); 
	//// 定义
	simpleBlock = ^{
        NSLog(@"This is a block");
    };
    //// 声明、定义一起
    void (^simpleBlock)(void) = ^{
        NSLog(@"This is a block");
    };
    // 调用 block
    simpleBlock();

###带参数和返回值的Block
下面的Block，接受两个double参数，返回一个double值：

	// 声明
	double (^multiplyTwoValues)(double, double);
	// 定义
	double (^multiplyTwoValues)(double, double) =
                              ^(double firstValue, double secondValue) {
                                  return firstValue * secondValue;
                              };
	// 调用
    double result = multiplyTwoValues(2,4);
    NSLog(@"The result is %f", result);

###Block的缩写
Block没有参数，则 `()` 可以省略：

	[UIView animateViewDuration:5.0 animation:^() { // 这个block没有参数，这里的 () 就可以省略。
		view.opacity = 0.5;
	}];

如果Block中返回值的类型根据return后的内容能明显推出，那么可以省略：

	NSSet* mySet = ...;
	NSSet* matches = [mySet objectsPassingTest:^BOOL(id obj, ...) { // 根据return语句，这个block的返回值明显是BOOL，所以这里可以省略BOOL。
		return [obj isKindOfClass:[UIView class]]; // 返回值是 BOOL，很明显。
	}];

###Block获得作用域的变量

- 局部变量（local varible）在 block 里是只读的。如果想在 block 中读写局部变量，那么需要在局部变量前加 __block。__block 修饰的局部变量在 block 中会被编译器从栈上传递到堆上，这样 block 就能使用它改变它，当 block 代码结束，block 会再把这个变量值拷贝回堆，再拷贝回栈上。

>
	// 不能改变局部变量：
	int anInteger = 42;
    void (^testBlock)(void) = ^{
        NSLog(@"Integer is: %i", anInteger);
    };
    anInteger = 84;
    testBlock(); // 输出：Integer is: 42
    // 加了 __block 后可以改变局部变量：
	__block int anInteger = 42;
    void (^testBlock)(void) = ^{
        NSLog(@"Integer is: %i", anInteger);
    };
    anInteger = 84;
    testBlock(); // 输出：Integer is: 84
    // 还可以这样：
	__block int anInteger = 42;
    void (^testBlock)(void) = ^{
        NSLog(@"Integer is: %i", anInteger);
        anInteger = 100;
    };
    testBlock(); // 输出：Integer is: 42
    NSLog(@"Value of original variable is now: %i", anInteger); // 输出：Value of original variable is now: 100

>
如果获得对象（在堆上），调用变更该对象的方法是没问题的（存储对象的空间在堆上），而直接向截获的变量赋值则会产生编译错误（这个对象的指针是在栈上的）。
>
	// 这样是没问题的：
	id array = [[NSMutableArray alloc] init];
	void (^blk)(void) = ^{
		id obj = [[NSObject alloc] init];
		[array addObject:obj];
	};
	// 这样是不对的：
	id array = [[NSMutableArray alloc] init];
	void (^blk)(void) = ^{
		array = [[NSMutableArray alloc] init];
	};

>
在 block 中使用 C 语言的数组时必须小心使用其指针。在现在的 block 中，截获自动变量的方法并没有实现对 C 语言数组的截获，使用指针可以解决该问题。
>
	// 这样是有问题的：
	const char text[] = "hello";
	void (^blk)(void) = {
		printf("%c\n", text[2]);
	}
	// 这样是没问题的：
	const char* text = "hello";
	void (^blk)(void) = {
		printf("%c\n", text[2]);
	}


- 实例变量 instance varible，在 block 里是可读可写的。

- 静态变量在 block 里是可读可写的。

- 全局变量在 block 里是可读可写的。


###把Block传递给方法或函数

	// 调用带block参数的方法：
	- (IBAction)fetchRemoteInformation:(id)sender {
	    [self showProgressIndicator];
	 
	    XYZWebTask *task = ...
	 
	    [task beginTaskWithCallbackBlock:^{
	        [self hideProgressIndicator];
	    }];
	}
	
	// 声明带block参数的方法：
	- (void)beginTaskWithCallbackBlock:(void (^)(void))callbackBlock;
	
	// 方法的具体实现：
	- (void)beginTaskWithCallbackBlock:(void (^)(void))callbackBlock {
    	...
    	callbackBlock();
	}

苹果的建议是在一个方法中最好只使用一个block变量，并且如果这个方法如果还带有其他非block变量，那么block变量应该放在最后一个。

###使用typedef来简化Block定义

	// typedef一个block
	typedef void (^XYZSimpleBlock)(void);
    // 使用1
	XYZSimpleBlock anotherBlock = ^{
        ...
    };
    // 使用2
    - (void)beginFetchWithCallbackBlock:(XYZSimpleBlock)callbackBlock {
    	...
    	callbackBlock();
	}
    
来看一个用typedef简化复杂Block定义的例子，下面的定义的名为complexBlock的变量是一个block，这个block接受一个block作为参数，并且返回一个block：

	// 简化前：
	void (^(^complexBlock)(void (^)(void)))(void) = ^ (void (^aBlock)(void)) {
    	...
    	return ^{
        	...
    	};
	};
	// 使用上面typedef的XYZSimpleBlock简化后：
	XYZSimpleBlock (^betterBlock)(XYZSimpleBlock) = ^ (XYZSimpleBlock aBlock) {
	    ...
	    return ^{
	        ...
	    };
	};

###定义属性来持有Block
使用`copy`，因为Block要持有它原本所在作用域的其他外面的变量：

	@interface XYZObject : NSObject
	@property (copy) void (^blockProperty)(void);
	@end

	// setter方法和调用
	self.blockProperty = ^{
        ...
    };
    self.blockProperty();
    
    // 使用typedef简化
    typedef void (^XYZSimpleBlock)(void);
 
	@interface XYZObject : NSObject
	@property (copy) XYZSimpleBlock blockProperty;
	@end

##使用Block需要注意的问题

###避免强引用循环
每次向 block 里的对象发送消息（方法调用）的时候，将会创建一个 strong 指针指向这个对象，直到 block 结束。所以像下面的代码，self strong 持有 block，而在 block 里又 strong 持有了 self，这样谁也不能被释放：

	// .h
	@interface XYZBlockKeeper : NSObject
	@property (copy) void (^block)(void);
	@end
	// .m
	@implementation XYZBlockKeeper
	- (void)configureBlock {
	    self.block = ^{
	    	// capturing a strong reference to self, creates a strong reference cycle
	        [self doSomething];
	    };
	}
	...
	@end

可以这样处理：

	- (void)configureBlock {
	    XYZBlockKeeper * __weak weakSelf = self;
	    self.block = ^{
	    	// capture the weak reference to avoid the reference cycle
	        [weakSelf doSomething];
	    }
	}


###避免Block使用的对象被提前释放
上面提到了使用 weak 引用的方式解决 retain cycle 的方案，但随着应用场景的改变，又会带来新的问题，比如：在 block 中用异步的方式使用了外部对象，当对象被释放后，异步方法回调时访问该对象则会为空，这时就可能造成程序崩溃了。解决这个问题的方式则是 weak/strong 化，示例代码如下：

TestBlockViewController.m

	#import "TestBlockViewController.h"
	#import "TestService.h"
	
	@interface TestBlockViewController ()
	@property (nonatomic, strong) NSString *tag;
	@property (nonatomic, copy) void (^myBlock)(void);
	@end
	
	@implementation TestBlockViewController
	
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    
	    // Init properties.
	    self.tag = @"tag is OK.";
	    
	    // Init TestService's block.
        __weak typeof(self)weakSelf = self;
	    self.myBlock = ^{
	        __strong typeof(weakSelf)strongSelf = weakSelf;
	        
	        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
                NSLog(@"strongSelf is OK.");
                NSLog(@"%@", strongSelf.tag);
                //NSLog(@"%@", self.tag); // Retain cycle.
	        });
	    };
	    
	}
	
	- (IBAction)callService:(id)sender {    
	    self.myBlock();
	    [self.navigationController popViewControllerAnimated:YES];
	}
	
	- (void)dealloc {
	    NSLog(@"TestBlockViewController dealloc.");
	}
	
	@end

这样即使当前的 TestBlockViewController 被 pop 后，block 中也由于强引用了 weakSelf 所以延长了其生命周期，最后打印的结果是：

	strongSelf is OK.
	tag is OK.
	TestBlockViewController dealloc.


##Block的使用场景

###Block简化枚举

	// NSArray的一个方法，接受一个block作为参数，这个block会在该方法里每枚举一个对象时被调用一次：
	- (void)enumerateObjectsUsingBlock:(void (^)(id obj, NSUInteger idx, BOOL *stop))block;
	// 这个block接受三个参数，前两个是当前的对象和其index，后面的BOOL值可以用来控制什么时候停止该枚举。
	NSArray *array = ...
    [array enumerateObjectsUsingBlock:^ (id obj, NSUInteger idx, BOOL *stop) {
        NSLog(@"Object at index %lu is %@", idx, obj);
    }];
    [array enumerateObjectsUsingBlock:^ (id obj, NSUInteger idx, BOOL *stop) {
        if (...) {
            *stop = YES;
        }
    }];

###Block简化并发调用
####Block和Operation Queue一起用

	NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
	    ...
	}];

####Block和GCD一起用

	dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
	dispatch_async(queue, ^{
	    NSLog(@"Block for asynchronous execution");
	});

###Block的使用场景小结

* Enumeration (like we saw above with NSArray)
* View Animations (animations)
* Sorting (sort this thing using a block as the comparison method)
* Notification (when something happens, execute this block)
* Error handlers (if an error happens while doing this, execute this block)
* Completion handlers (when you are done doing this, execute this block)
* Multithreading (With Grand Central Dispatch (GCD) API)


[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html