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


Block能够让我们创建明显的代码片段，并且可以像参数那样传递给方法或函数。在Objective-C中，Block就像对象一样，能够被添加到集合中（比如：NSArray、NSDictionary）。Block能够获得其所在作用域的变量，就如同其他语言里`闭包（Closure）`或者`lambda计算`的概念。

##Block语法形式
Block 的语法遵循如下形式：

- `^` `返回值类型` `参数列表` `表达式`

其中`返回值类型`是可以省略的，但需要保证`表达式`中的所有 return 语句的返回值类型相同。`参数列表`在为空的情况下也是可以省略的。

一些示例：

	// 一个最简单的block：
	^{
    	NSLog(@"This is a block");
    }
    
    // 不省略的block：
    ^void (void) {
    	NSLog(@"This is a block");
    }
    
    // 普通的block：
    ^int (int a, int b) {
    	NSLog(@"a is %d, b is %d", a, b);
    	return a + b;
    }
    
可见，Block 语法从形式上来看除了没有名字以及带有 `^` 外，其他都和 C 语言的函数定义相同。


##Block类型变量的使用
在 C 语言中，可以把一个函数的地址赋值给函数指针类型的变量，如下：

	int func(int a) {
		return a + 1;
	}
	int (*func_pointer)(int) = &func;
	
同样，在 Block 语法中，也可以把一个 Block 赋值给 Block 类型的变量。

###最简单的Block
声明和定义一个不返回任何值，不接受任何参数的 Block：
    
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

###Block作为参数时的缩写

Block 作为参数时的缩写如 Block 语法规则所约定那样。

	// Block没有参数，则 `()` 可以省略：
	[UIView animateViewDuration:5.0 animation:^() { // 这个block没有参数，这里的 () 就可以省略。
		view.opacity = 0.5;
	}];

	// 如果Block中返回值的类型根据return后的内容能明显推出，那么可以省略：
	NSSet* mySet = ...;
	NSSet* matches = [mySet objectsPassingTest:^BOOL(id obj, ...) { // 根据return语句，这个block的返回值明显是BOOL，所以这里可以省略BOOL。
		return [obj isKindOfClass:[UIView class]]; // 返回值是 BOOL，很明显。
	}];



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

上面的 Block 类型的变量在使用时，记述方式会一眼看上去有点不够简洁，这时候我们也可以用 typedef 来解决这个问题。

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

###Block的指针类型变量
我们还可以使用指向 Block 类型变量的指针，即 Block 的指针类型变量。

	typedef int (^MyBlock)(int);
	MyBlock aBlock = ^int(int a) {
		return a + 1;
	};
	
	MyBlock* aBlockPointer = &aBlock;
	int result = (*aBlockPointer)(10); // result = 11.
	
可见 Block 类型变量可像 C 语言中其他类型变量一样使用。

##使用Block需要注意的问题

###Block截获作用域的变量

- 局部变量（local varible）在 block 里是只读的。如果想在 block 中读写局部变量，那么需要在局部变量前加 `__block`。`__block` 修饰的局部变量在 block 中会被编译器从栈上传递到堆上，这样 block 就能使用它改变它，当 block 代码结束，block 会再把这个变量值拷贝回堆，再拷贝回栈上。

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


##Block的实现
在前面的内容中，我们知道了 Block 是「能持有作用域变量的匿名函数」，还介绍了使用 Block 的相关内容，那么 Block 究竟是如何实现的呢？我们可以用 clang（LLVM 编译器）把带 Block 语法的源代码代码转换为我们能够理解的源代码来初探一下。这里我们可以使用 `clang -rewrite-objc <source-code-file>` 把含有 Block 语法的源代码转换成 C++ 的源代码（这里其实就是使用了 struct 结构的 C 代码）。 

包含 Block 语法的源代码 test_block.m：

	#include <stdio.h>
	
	int main() {
		void (^blk)(void) = ^{
			int tag = 8;
			printf("Block, %d\n", tag);
		};
	
		blk();
	
		return 0;
	}
	
	
使用 `clang -rewrite-objc test_block.m` 编译后得到了 test_block.cpp 代码，从里面截取相关的代码如下：

	struct __block_impl {
		void *isa;
		int Flags;
		int Reserved;
		void *FuncPtr;
	};

	struct __main_block_impl_0 {
		struct __block_impl impl;
		struct __main_block_desc_0* Desc;
		__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
			impl.isa = &_NSConcreteStackBlock;
			impl.Flags = flags;
			impl.FuncPtr = fp;
			Desc = desc;
		}
	};
	
	// 这里对应的就是 block 的匿名函数。参数 __cself 为指向 block 值的指针，类似 Objective-C 中实例方法中指向对象自身的变量 self。
	static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
		int tag = 0;
		printf("Block, %d\n", tag);
	 }
	
	static struct __main_block_desc_0 {
		size_t reserved;
		size_t Block_size;
	} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

	int main() {
		void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA);
	
		((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
	
		return 0;
	}

如转换后的代码所示，Block 使用的匿名代码被转换为了简单的 C 语言函数，其函数名则根据 Block 语法所属的函数名（这里是 main）和该 Block 语法在该函数出现的顺序值（这里是 0）来命名。

=== 未完待续 ===


[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html