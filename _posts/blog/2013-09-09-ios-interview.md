---
layout: post
title: iOS 面试知识集锦
description: 攒一些面试题。
category: blog
tag: iOS, Objective-C, Swift
---

1、什么情况使用 weak 关键字，相比 assign 有什么不同？

什么情况使用 weak 关键字？

- 在 ARC 中，在有可能出现循环引用的时候，往往要通过让其中一端使用 weak 来解决，比如: delegate、block。
- 自身已经对它进行一次强引用，没有必要再强引用一次，此时也会使用 weak，自定义 IBOutlet 控件属性一般也使用 weak，使用 storyboard（xib 不行）创建的 vc，会有一个叫 `_topLevelObjectsToKeepAliveFromStoryboard` 的私有数组强引用所有 top level 的对象，所以这时即便 outlet 声明成 weak 也没关系。当然，也可以使用 strong。

weak 和 assign 的不同点：

- weak、assign 修饰的属性指向一个对象时都不会增加对象的引用计数。然而在所指的对象被释放时，weak 属性值会被置为 nil，而 assign 属性不会。
- assign 可以用非 OC 对象以及基本类型，而 weak 必须用于 OC 对象。



2、怎么用 copy 关键字？

copy 的语义是将对象拷贝一份给新的引用，通过新的引用对它的修改不影响原来那个被拷贝的对象。

- NSString、NSArray、NSDictionary 等等经常使用 copy 关键字，是因为他们有对应的可变类型：NSMutableString、NSMutableArray、NSMutableDictionary。
- block 也经常使用 copy 关键字。block 使用 copy 是从 MRC 遗留下来的传统，在 MRC 中，方法内部的 block 是在栈区的，使用 copy 可以把它放到堆区。在 ARC 中写不写都行，对于 block 使用 copy 还是 strong 效果是一样的，但写上 copy 也无伤大雅，还能时刻提醒我们：编译器自动对 block 进行了 copy 操作。



3、用 @property 声明的 NSString（或 NSArray，NSDictionary）经常使用 copy 关键字，为什么？如果改用 strong 关键字，可能造成什么问题？

- 使用 copy 无论给我传入是一个可变对象还是不可对象，我本身持有的就是一个不可变的副本。
- 如果使用 strong，那么这个属性就有可能指向一个可变对象，如果这个可变对象在外部被修改了，那么会影响该属性。

```
@property (nonatomic, readwrite, strong) NSArray *myArray;

NSArray *array = @[@1, @2, @3, @4];
NSMutableArray *mutableArray = [NSMutableArray arrayWithArray:array];

self.myArray = mutableArray;
[mutableArray removeAllObjects];;
NSLog(@"%@", self.myArray); // ()

[mutableArray addObjectsFromArray:array];
self.myArray = [mutableArray copy];
[mutableArray removeAllObjects];;
NSLog(@"%@", self.myArray); // (1,2,3,4)
```



4、怎么理解浅拷贝与深拷贝？

不论是非集合类对象还是集合类对象：

- copy 返回的是 imutable 对象；所以，如果对 copy 返回值使用 mutable 对象接口就会 crash。
- mutableCopy 返回 mutable 对象。

对非集合类对象：

- [immutableObject copy] // 浅复制
- [immutableObject mutableCopy] // 深复制
- [mutableObject copy] // 深复制
- [mutableObject mutableCopy] // 深复制

对集合类对象：

- [immutableObject copy] // 浅复制
- [immutableObject mutableCopy] // 单层深复制
- [mutableObject copy] // 单层深复制
- [mutableObject mutableCopy] // 单层深复制

浅复制(shallow copy)：在浅复制操作时，对于被复制对象的每一层都是指针复制。
单层深复制(one-level-deep copy)：在单层深复制操作时，对于被复制对象，至少有一层是深复制。
深复制(real-deep copy)：在深复制操作时，对于被复制对象的每一层都是对象复制。

参考：[iOS 集合的深复制与浅复制](https://www.zybuluo.com/MicroCai/note/50592)



5、如何让自己的类用 copy 修饰符？

想让自己所写的对象具有拷贝功能，则需实现 NSCopying 协议。如果自定义的对象分为可变版本与不可变版本，那么就要同时实现 NSCopying 与 NSMutableCopying 协议。

实现 NSCopying 协议。该协议只有一个方法：`- (id)copyWithZone:(NSZone *)zone;`。

实现 NSMutableCopying 协议。该协议只有一个方法：`- (id)mutableCopyWithZone:(nullable NSZone *)zone;`



6、@property 的本质是什么？

@property = ivar + getter + setter;

属性(property)有两大概念：ivar（实例变量）、存取方法（access method ＝ getter + setter）。



7、@protocol 和 category 中如何使用 @property？

在 protocol 中使用 property 只会生成 setter 和 getter 方法声明，我们使用属性的目的，是希望遵守我协议的对象能实现该属性。

category 使用 @property 也是只会生成 setter 和 getter 方法的声明，如果我们真的需要给 category 增加属性的实现，需要借助于运行时的两个函数：

- objc_setAssociatedObject
- objc_getAssociatedObject


8、runtime 如何实现 weak 属性？

weak 此特质表明该属性定义了一种「非拥有关系」(nonowning relationship)。为这种属性设置新值时，设置方法既不持有新值，也不释放旧值。

runtime 对注册的类，会进行内存布局，对于 weak 对象会放入一个 hash 表中。用 weak 指向的对象内存地址作为 key，当此对象的引用计数为 0 的时候会 dealloc，假如 weak 指向的对象内存地址是 a，那么就会以 a 为键，在这个 weak 表中搜索，找到所有以 a 为键的 weak 对象，从而设置为 nil。



9、@synthesize 和 @dynamic 分别有什么作用？

- @property 有两个对应的词，一个是 @synthesize，一个是 @dynamic。如果 @synthesize 和 @dynamic 都没写，那么默认的就是 `@syntheszie var = _var;`。
- @synthesize 的语义是如果你没有手动实现 setter 方法和 getter 方法，那么编译器会自动为你加上这两个方法。
- @dynamic 告诉编译器：属性的 setter 与 getter 方法由用户自己实现，不自动生成。（当然对于 readonly 的属性只需提供 getter 即可）。假如一个属性被声明为 @dynamic var，然后你没有提供 @setter 方法和 @getter 方法，编译的时候没问题，但是当程序运行到 instance.var = someVar，由于缺 setter 方法会导致程序崩溃；或者当运行到 someVar = var 时，由于缺 getter 方法同样会导致崩溃。编译时没问题，运行时才执行相应的方法，这就是所谓的动态绑定。



10、ARC 下，不显式指定任何属性关键字时，默认的关键字都有哪些？

对应基本数据类型默认关键字是：atomic, readwrite, assign。

对于普通的 Objective-C 对象默认关键字是：atomic, readwrite, strong。



11、在有了自动合成属性实例变量之后，@synthesize 还有哪些使用场景？


总结下 @synthesize 合成实例变量的规则，有以下几点：

- 如果指定了成员变量的名称，会生成一个指定的名称的成员变量 `@synthesize foo = _foo;`。如果这个成员已经存在了就不再生成了。
- 如果是 `@synthesize foo;` 会生成一个名称为 foo 的成员变量，也就是说：如果没有指定成员变量的名称会自动生成一个属性同名的成员变量，
- 假如 property 名为 foo，同时还存在一个名为 `_foo` 的实例变量，则不会自动合成新变量。


回答这个问题前，我们要搞清楚一个问题：什么情况下不会 autosynthesis（自动合成）？

- 同时重写了 setter 和 getter 时
- 重写了只读属性的 getter 时
- 使用了 @dynamic 时
- 在 @protocol 中定义的所有属性
- 在 category 中定义的所有属性
- 重载的属性



12、一个 objc 对象如何进行内存布局（考虑有父类的情况）？

- 每一个对象内部都有一个 isa 指针，指向他的类对象，类对象中存放着本对象的：
	- 对象方法列表（对象能够接收的消息列表，保存在它所对应的类对象中）。
	- 成员变量的列表。
	- 属性列表。
	- 类对象内部也有一个 isa 指针指向元对象(meta class)，元对象内部存放的是类方法列表。
	- 类对象内部还有一个 superclass 的指针，指向他的父类对象。
- 所有父类的成员变量和自己的成员变量都会存放在该对象所对应的存储空间中。


Objective-C 对象的结构图：
	- isa 指针
	- 根类的实例变量
	- 倒数第二层父类的实例变量
	- ...
	- 父类的实例变量
	- 类的实例变量

![image](../../images/ios-interview/instance-structure.png)


13、runtime 如何通过 selector 找到对应的 IMP 地址（分别考虑类方法和实例方法）？

每一个类对象中都一个方法列表，方法列表中记录着方法的名称，方法实现，以及参数类型，其实 selector 本质就是方法名称，通过这个方法名称就可以在方法列表中找到对应的方法实现。



14、objc 中的类方法和实例方法有什么本质区别和联系？

类方法：

- 类方法是属于类对象的
- 类方法只能通过类对象调用
- 类方法中的 self 是类对象
- 类方法可以调用其他的类方法
- 类方法中不能访问成员变量
- 类方法中不能直接调用对象方法

实例方法：

- 实例方法是属于实例对象的
- 实例方法只能通过实例对象调用
- 实例方法中的 self 是实例对象
- 实例方法中可以访问成员变量
- 实例方法中直接调用实例方法
- 实例方法中也可以调用类方法（通过类名）



15、`_objc_msgForward` 函数是做什么的？

`_objc_msgForward` 是 IMP 类型（函数指针），用于消息转发的：当向一个对象发送一条消息，但它并没有实现的时候，`_objc_msgForward` 会尝试做消息转发。

在消息传递过程中，objc_msgSend 的动作比较清晰：首先在 Class 中的缓存查找 IMP （没缓存则初始化缓存），如果没找到，则向父类的 Class 查找。如果一直查找到根类仍旧没有实现，则用 `_objc_msgForward` 函数指针代替 IMP。最后，执行这个 IMP。

当调用一个 NSObject 对象不存在的方法时，并不会马上抛出异常，而是会经过多层转发，层层调用对象的 `-resolveInstanceMethod:`, `-forwardingTargetForSelector:`, `-methodSignatureForSelector:`, `-forwardInvocation:` 等方法。其中最后 `-forwardInvocation:` 是会有一个 NSInvocation 对象，这个 NSInvocation 对象保存了这个方法调用的所有信息，包括 Selector 名，参数和返回值类型，最重要的是有所有参数值，可以从这个 NSInvocation 对象里拿到调用的所有参数值。我们可以想办法让每个需要被 JS 替换的方法调用最后都调到 `-forwardInvocation:`，就可以解决无法拿到参数值的问题了。

JSPatch 实现 hotpatch 具体实现，以替换 UIViewController 的 `-viewWillAppear:` 方法为例：

- 把 UIViewController 的 `-viewWillAppear:` 方法通过 `class_replaceMethod()` 接口指向一个不存在的 IMP: `class_getMethodImplementation(cls, @selector(__JPNONImplementSelector))`，这样调用这个方法时就会走到 `-forwardInvocation:`。
- 为 UIViewController 添加 `-ORIGviewWillAppear:` 和 `-_JPviewWillAppear:` 两个方法，前者指向原来的 IMP 实现，后者是新的实现，稍后会在这个实现里回调 JS 函数。
- 改写 UIViewController 的 `-forwardInvocation:` 方法为自定义实现。一旦 OC 里调用 UIViewController 的 `-viewWillAppear:` 方法，经过上面的处理会把这个调用转发到 `-forwardInvocation:`，这时已经组装好了一个 NSInvocation，包含了这个调用的参数。在这里把参数从 NSInvocation 反解出来，带着参数调用上述新增加的方法 `-JPviewWillAppear:`，在这个新方法里取到参数传给 JS，调用 JS 的实现函数。整个调用过程就结束了。



16、能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？

- 不能向编译后得到的类中增加实例变量。
- 能向运行时创建的类中添加实例变量。

解释下：

- 因为编译后的类已经注册在 runtime 中，类结构体中的 objc_ivar_list 实例变量的链表和 instance_size 实例变量的内存大小已经确定，同时 runtime 会调用 class_setIvarLayout 或 class_setWeakIvarLayout 来处理 strong weak 引用。所以不能向存在的类中添加实例变量。
- 运行时创建的类是可以添加实例变量，调用 class_addIvar 函数。但是得在调用 objc_allocateClassPair 之后，objc_registerClassPair 之前，原因同上。



17、run loop 和线程有什么关系？

总的说来，run loop，正如其名，loop 表示某种循环，和 run 放在一起就表示一直在运行着的循环。实际上，run loop 和线程是紧密相连的，可以这样说 run loop 是为了线程而生，没有线程，它就没有存在的必要。run loop 是线程的基础架构部分，Cocoa 和 CoreFundation 都提供了 run loop 对象方便配置和管理线程的 run loop（以下都以 Cocoa 为例）。每个线程，包括程序的主线程（main thread）都有与之相应的 run loop 对象。

- 主线程的 run loop 默认是启动的。

```
int main(int argc, char * argv[]) {
   @autoreleasepool {
       return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
   }
}
```

重点是 UIApplicationMain() 函数，这个方法会为 main thread 设置一个 NSRunLoop 对象，这就解释了：为什么我们的应用可以在无人操作的时候休息，需要让它干活的时候又能立马响应。

- 对其它线程来说，run loop 默认是没有启动的，如果你需要更多的线程交互则可以手动配置和启动，如果线程只是去执行一个长时间的已确定的任务则不需要。

- 在任何一个 Cocoa 程序的线程中，都可以通过以下代码来获取到当前线程的 run loop。
	- `NSRunLoop *runloop = [NSRunLoop currentRunLoop];`




18、run loop 的 mode 作用是什么？

model 主要是用来指定事件在运行循环中的优先级的，分为：

- NSDefaultRunLoopMode（kCFRunLoopDefaultMode）：默认，空闲状态
- UITrackingRunLoopMode：ScrollView滑动时
- UIInitializationRunLoopMode：启动时
- NSRunLoopCommonModes（kCFRunLoopCommonModes）：Mode集合

苹果公开提供的 Mode 有两个：

- NSDefaultRunLoopMode（kCFRunLoopDefaultMode）
- NSRunLoopCommonModes（kCFRunLoopCommonModes）



19、以 `+ scheduledTimerWithTimeInterval...` 的方式触发的 timer，在滑动页面上的列表时，timer 会暂定回调，为什么？如何解决？


RunLoop 只能运行在一种 mode 下，如果要换 mode，当前的 loop 也需要停下重启成新的。利用这个机制，ScrollView 滚动过程中 NSDefaultRunLoopMode（kCFRunLoopDefaultMode）的 mode 会切换到 UITrackingRunLoopMode 来保证 ScrollView 的流畅滑动：只能在 NSDefaultRunLoopMode 模式下处理的事件会影响 ScrollView 的滑动。

如果我们把一个 NSTimer 对象以 NSDefaultRunLoopMode（kCFRunLoopDefaultMode）添加到主运行循环中的时候, ScrollView 滚动过程中会因为 mode 的切换，而导致 NSTimer 将不再被调度。

Timer 计时会被 scrollView 的滑动影响的问题可以通过将 timer 添加到 NSRunLoopCommonModes（kCFRunLoopCommonModes）来解决。

```
// 默认情况：将 timer 添加到 NSDefaultRunLoopMode 中：
[NSTimer scheduledTimerWithTimeInterval:1.0
     target:self
     selector:@selector(timerTick:)
     userInfo:nil
     repeats:YES];

// 手动将 timer 添加到 NSRunLoopCommonModes 里：
NSTimer *timer = [NSTimer timerWithTimeInterval:1.0
     target:self
     selector:@selector(timerTick:)
     userInfo:nil
     repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```



20、猜想 run loop 内部是如何实现的？

一般来讲，一个线程一次只能执行一个任务，执行完成后线程就会退出。如果我们需要一个机制，让线程能随时处理事件但并不退出，通常的代码逻辑 是这样的：

```
function loop() {
    initialize();
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
```



21、objc 使用什么机制管理对象内存？

通过 retainCount 的机制来决定对象是否需要释放。 每次 run loop 的时候，都会检查对象的 retainCount，如果 retainCount 为 0，说明该对象没有地方需要继续使用了，可以释放掉了。



22、ARC 通过什么方式帮助开发者管理内存？

ARC 相对于 MRC，不是在编译时添加 retain/release/autorelease 这么简单。应该是编译期和运行期两部分共同帮助开发者管理内存。

在编译期，ARC 用的是更底层的 C 接口实现的 retain/release/autorelease，这样做性能更好，也是为什么不能在 ARC 环境下手动 retain/release/autorelease，同时对同一上下文的同一对象的成对 retain/release 操作进行优化（即忽略掉不必要的操作）；ARC 也包含运行期组件，这个地方做的优化比较复杂，但也不能被忽略。



23、一个 autorealese 对象在什么时刻释放？

分两种情况：手动干预释放时机、系统自动去释放。

- 手动干预释放时机：手动指定 autoreleasepool 的 autorelease 对象，在当前作用域大括号结束时释放。
- 系统自动去释放：不手动指定 autoreleasepool 的 autorelease 对象出了作用域之后，会被添加到最近一次创建的自动释放池中，并会在当前的 runloop 迭代结束时释放。而它能够释放的原因是系统在每个 runloop 迭代中都加入了自动释放池 Push 和 Pop。

例子：

```
__weak id reference = nil;
- (void)viewDidLoad {
    [super viewDidLoad];
    NSString *str = [NSString stringWithFormat:@"aaa"];
    // str 是一个 autorelease 对象，设置一个 weak 的引用来观察它。
    reference = str;
}
- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];
    NSLog(@"%@", reference); // Console: aaa
}
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    NSLog(@"%@", reference); // Console: (null)
}
```

由于这个 vc 在 loadView 之后便 add 到了 window 层级上，所以 viewDidLoad 和 viewWillAppear 是在同一个 runloop 调用的，因此在 viewWillAppear 中，这个 autorelease 的变量依然有值。而在 viewDidAppear 执行之前这个 autorelease 的变量已经被释放了。



从程序启动到加载完成是一个完整的 runloop，然后会停下来，等待用户交互，用户的每一次交互都会启动一次运行循环，来处理用户所有的点击事件、触摸事件。

我们都知道：所有 autorelease 的对象，在出了作用域之后，会被自动添加到最近创建的自动释放池中。

但是如果每次都放进应用程序的 main.m 中的 autoreleasepool 中，迟早有被撑满的一刻。所以在每一次完整的 runloop 结束之前，对于的自动释放池里面的 autorelease 对象会被销毁。那这个自动释放池是什么时候创建的呢？答案是，在 runloop 检测到事件并启动后，就会创建对应的自动释放池。


子线程的 runloop 默认是不工作，无法主动创建，必须手动创建。

自定义的 NSOperation 和 NSThread 需要手动创建自动释放池。比如： 自定义的 NSOperation 类中的 main 方法里就必须添加自动释放池。否则出了作用域后，自动释放对象会因为没有自动释放池去处理它，而造成内存泄露。但对于 blockOperation 和 invocationOperation 这种默认的 Operation ，系统已经帮我们封装好了，不需要手动创建自动释放池。

@autoreleasepool 当自动释放池被销毁或者耗尽时，会向自动释放池中的所有对象发送 release 消息，释放自动释放池中的所有对象。



24、如何实现 autoreleasepool 的？


autoreleasepool 以一个队列数组的形式实现，主要通过下列三个函数完成.

- objc_autoreleasepoolPush
- objc_autoreleasepoolPop
- objc_autorelease



25、如何用 GCD 同步若干个异步调用？

使用 Dispatch Group 追加 block 到 Global Group Queue，这些 block 如果全部执行完毕，就会执行 Main Dispatch Queue 中的结束处理的 block。

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
dispatch_group_async(group, queue, ^{ /*加载图片1 */ });
dispatch_group_async(group, queue, ^{ /*加载图片2 */ });
dispatch_group_async(group, queue, ^{ /*加载图片3 */ }); 
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    // 合并图片
});
```



26、dispatch_barrier_async 的作用是什么？

dispatch_barrier_async 函数配合 Concurrent Dispatch Queue 一起使用可以在并行的任务中插入中间任务。

```
dispatch_queue_t queue = dispatch_queue_create("com.example.gcd.ForBarrier", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(queue, blk0_for_reading);
dispatch_async(queue, blk1_for_reading);
dispatch_async(queue, blk2_for_reading);
dispatch_async(queue, blk3_for_reading);
dispatch_barrier_async(queue, blk_for_writing);
dispatch_async(queue, blk4_for_reading);
dispatch_async(queue, blk5_for_reading);
dispatch_async(queue, blk6_for_reading);
dispatch_async(queue, blk7_for_reading);
```

dispatch_barrier_async 函数会等待当前 Concurrent Dispatch Queue 中并行执行的读取任务(blk0-3_for_reading)都结束后，再将指定的 blk_for_writing 任务添加到 Concurrent Dispatch Queue 中，然后只有在这个任务执行完毕后，后面添加到 Concurrent Dispatch Queue 的任务(blk4-7_for_reading)才恢复正常的并行执行的模式。可见，Concurrent Dispatch Queue 和 dispatch_barrier_async 搭配使用可以使编码非常清晰，同时可以实现高效率的数据库访问和文件访问。



27、苹果为什么要废弃 dispatch_get_current_queue？

dispatch_get_current_queue 容易造成死锁。

```
- (void)viewDidLoad
{
    [super viewDidLoad];
    NSLog(@"1");
    dispatch_sync(dispatch_get_main_queue(), ^{
        NSLog(@"2");
    });
    NSLog(@"3");
}
```

只输出：1。发生主线程锁死。

dispatch_sync 函数用于将一个 block 提交到队列中同步执行，同步（sync）操作会阻塞当前线程并等待 block 中的任务执行完毕才会返回。dispatch_get_main_queue() 得到的是一个串行队列，串行队列的特点：一次只调度一个任务，队列中的任务一个接着一个地执行（一个任务执行完毕后，再执行下一个任务）。所以就锁死了。



28、如何手动触发一个 value 的 KVO？

KVC，即是指 NSKeyValueCoding，一个非正式的 Protocol，提供一种机制来间接访问对象的属性。KVO 就是基于 KVC 实现的关键技术之一。

键值观察通知依赖于 NSObject 的两个方法: `willChangeValueForKey:` 和 `didChangevlueForKey:`。在一个被观察属性发生改变之前，`willChangeValueForKey:` 一定会被调用，这就会记录旧的值。而当改变发生后，`observeValueForKey:ofObject:change:context:` 和 `didChangeValueForKey:` 也会被调用。如果可以手动实现这些调用，就可以实现“手动触发”了。

```
@property (nonatomic, strong) NSDate *now;

- (void)viewDidLoad {
   [super viewDidLoad];
   _now = [NSDate date];
   [self addObserver:self forKeyPath:@"now" options:NSKeyValueObservingOptionNew context:nil];
   NSLog(@"1");
   [self willChangeValueForKey:@"now"]; // 手动触发 self.now 的 KVO，必写。
   NSLog(@"2");
   [self didChangeValueForKey:@"now"]; // 手动触发 self.now 的 KVO，必写。
   NSLog(@"4");
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context {
   NSLog(@"3");
}
```

打印顺序是：1 2 3 4。从这里看顺序似乎是 `wilChangeValueForKey:`、`observeValueForKeyPath:ofObject:change:context:`、`didChangeValueForKey:`。其实，实际情况是：`wilChangeValueForKey:` 先调用，接着是调用 `didChangeValueForKey:`，在 `didChangeValueForKey:` 内部调用了 `observeValueForKeyPath:ofObject:change:context:`。你可以注释掉 `[self didChangeValueForKey:@"now"];` 试试。

但是平时我们一般不会这么干，我们都是等系统去“自动触发”。“自动触发”的实现原理：

比如调用 setNow: 时，系统还会以某种方式在中间插入 `wilChangeValueForKey:`、`didChangeValueForKey:` 和 `observeValueForKeyPath:ofObject:change:context:` 的调用。

大致表现如下：

```
- (void)setNow:(NSDate *)aDate {
   [self willChangeValueForKey:@"now"];
   [super setValue:aDate forKey:@"now"];
   [self didChangeValueForKey:@"now"];
}
```

Apple 使用了 isa 混写（isa-swizzling）来实现 KVO，这种继承和方法注入是在运行时而不是编译时实现的。这就是正确命名如此重要的原因。只有在使用 KVC 命名约定时，KVO 才能做到这一点。KVO 在实现中通过 isa 混写（isa-swizzling）把这个对象的 isa 指针（isa 指针告诉 Runtime 系统这个对象的类是什么）指向这个新创建的子类，对象就神奇的变成了新创建的子类的实例。Apple 还重写、覆盖了 -class 方法并返回原来的类，企图欺骗我们：这个类没有变，就是原本那个类。


29、BAD_ACCESS 在什么情况下出现？

- 访问了野指针。比如对一个已经释放的对象执行了 release，访问已经释放对象的成员变量或者发消息。
- 死循环。


30、 如何调试 BAD_ACCESS 错误？

- 重写 object 的 respondsToSelector 方法，现实出现 EXEC_BAD_ACCESS 前访问的最后一个 object。
- 通过 Zombie。
- 设置全局断点快速定位问题代码所在行。
- Xcode 7 已经集成了 BAD_ACCESS 捕获功能：Address Sanitizer。用法如下：在配置中勾选 Enable Address Sanitizer。




[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/ios-interview