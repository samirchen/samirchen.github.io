---
layout: post
title: iOS 程序性能优化
description: 总结一下 iOS 程序性能优化相关的知识。
category: blog
tag: iOS, Objective-C, performance
---

##前言
程序性能优化不应该是一件放在功能完成之后的事，对性能的概念应该从我们一开始写代码时就萦绕在我们脑子里。了解 iOS 程序性能优化的相关知识点，从一开始就把它们落实到代码中是一种好的习惯。


##初级技巧

###使用复用机制
在我们使用 UITableView 和 UICollectionView 时我们通常会遇到「复用 Cell」这个提法，所谓「复用 Cell」就是指当需要展示的数据条目较多时，只创建较少数量的 Cell 对象（一般是屏幕可显示的 Cell 数再加一）并通过复用它们的方式来展示数据的机制。这种机制不会为每一条数据都创建一个 Cell，所以可以节省内存，提升程序的效率和交互流畅性。

从 iOS 6 以后，我们在 UITableView 和 UICollectionView 中不光可以复用 Cell，还可以复用各个 Section 的 Header 和 Footer。

在 UITableView 做复用的时候，会用到的 API：
	
	// 复用 Cell：
	- [UITableView dequeueReusableCellWithIdentifier:];
	- [UITableView registerNib:forCellReuseIdentifier:];
	- [UITableView registerClass:forCellReuseIdentifier:];
	- [UITableView dequeueReusableCellWithIdentifier:forIndexPath:];
	
	// 复用 Section 的 Header/Footer：
	- [UITableView registerNib:forHeaderFooterViewReuseIdentifier:];
	- [UITableView registerClass:forHeaderFooterViewReuseIdentifier:];
	- [UITableView dequeueReusableHeaderFooterViewWithIdentifier:];

复用机制是一个很好的机制，但是不正确的使用却会给我们的程序带来很多问题。下面拿 UITableView 复用 Cell 来举例：

	- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
	    static NSString *CellIdentifier = nil;
	    UITableViewCell *cell = nil;
	
	    CellIdentifier = @"UITableViewCell";
	    cell = [tableView dequeueReusableCellWithIdentifier:CellIdentifier];
	    if (!cell) {
	        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:CellIdentifier];
	        // 偶数行 Cell 的 textLabel 的文字颜色为红色。
	        if (indexPath.row % 2 == 0) {
	            [cell.textLabel setTextColor:[UIColor redColor]];
	        }
	    }
	    
	    cell.textLabel.text = @"Title";
	    // 偶数行 Cell 的 detailTextLabel 显示 Detail 文字。
	    if (indexPath.row % 2 == 0) {
	        cell.detailTextLabel.text = @"Detail";
	    }
	    
	    return cell;
	}

我们本来是希望只有偶数行的 textLabel 的文字颜色为红色，并且显示 Detail 文字，但是当你滑动 TableView 的时候发现不对了，有些奇数行的 textLabel 的文字颜色为红色，而且还显示了 Detail 文字，很奇怪。其实造成这个问题的原因就是「复用」，当一个 Cell 被拿来复用时，它所有被设置的属性（包括样式和内容）都会被拿来复用，如果刚好某一个的 Cell 你没有显式地设置它的属性，那么它这些属性就直接复用别的 Cell 的了。就如上面的代码中，我们并没有显式地设置奇数行的 Cell 的 textLabel 的文字颜色以及 detailTextLabel 的文字，那么它就有可能复用别的 Cell 的这些属性了。此外，还有个问题，对偶数行 Cell 的 textLabel 的文字颜色的设置放在了初始一个 Cell 的 if 代码块里，这样在复用的时候，逻辑走不到这里去，那么也会出现复用问题。所以，上面的代码需要改成这样：

	- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
	    static NSString *CellIdentifier = nil;
	    UITableViewCell *cell = nil;
	
	    CellIdentifier = @"UITableViewCell";
	    cell = [tableView dequeueReusableCellWithIdentifier:CellIdentifier];
	    if (!cell) {
	        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:CellIdentifier];
	    }
	    
	    cell.textLabel.text = @"Title";
	    
	    if (indexPath.row % 2 == 0) {
	        [cell.textLabel setTextColor:[UIColor redColor]];
	        cell.detailTextLabel.text = @"Detail";
	    }
	    else {
	        [cell.textLabel setTextColor:[UIColor blackColor]];
	        cell.detailTextLabel.text = nil;
	    }
	    
	    return cell;
	}

总之在复用的时候需要记住：

- **设置 Cell 的存在差异性的那些属性（包括样式和内容）时，有了 if 最好就要有 else，要显式的覆盖所有可能性。**
- **设置 Cell 的存在差异性的那些属性时，代码要放在初始化代码块的外部。**

上面的代码中，我们展示了 `- [UITableView dequeueReusableCellWithIdentifier:];` 的用法。下面看看另几个 API 的用法：

	@property (weak, nonatomic) IBOutlet UITableView *myTableView;
	
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    
	    // Setup table view.
	    self.myTableView.delegate = self;
	    self.myTableView.dataSource = self;
	    [self.myTableView registerClass:[MyTableViewCell class] forCellReuseIdentifier:@"MyTableViewCell"];
	}
	
	- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
	    static NSString *CellIdentifier = nil;
	    UITableViewCell *cell = nil;
	
	    CellIdentifier = @"MyTableViewCell";
	    cell = [tableView dequeueReusableCellWithIdentifier:CellIdentifier forIndexPath:indexPath];
	    
	    cell.textLabel.text = @"Title";
	    
	    if (indexPath.row % 2 == 0) {
	        [cell.textLabel setTextColor:[UIColor redColor]];
	    }
	    else {
	        [cell.textLabel setTextColor:[UIColor blackColor]];
	    }
	    
	    return cell;
	}

可以看到，`- [UITableView dequeueReusableCellWithIdentifier:forIndexPath:];` 必须搭配 `- [UITableView registerClass:forCellReuseIdentifier:];` 或者 `- [UITableView registerNib:forCellReuseIdentifier:];` 使用。当有可重用的 Cell 时，前者直接拿来复用，并调用 `- [UITableViewCell prepareForReuse]` 方法；当没有时，前者会调用 Identifier 对应的那个注册的 UITableViewCell 类的 `- [UITableViewCell initWithStyle:reuseIdentifier:]` 方法来初始化一个，这里省去了你自己初始化的步骤。当你自定义了一个 UITableViewCell 的子类时，你可以这样来用。


###尽可能设置 View 为不透明
UIView 有一个 `opaque` 属性，在你不需要透明效果时，你应该尽量设置它为 YES 可以提高绘图过程的效率。

在一个静态的视图里，这点可能影响不大，但是当在一个可以滚动的 Scroll View 中或是一个复杂的动画中，透明的效果可能会对程序的性能有较大的影响。

###避免臃肿的 XIB 文件
如果你压根不用 XIB，那就不需要看了。

在你需要重用某些自定义 View 或者因为历史兼容原因用到 XIB 的时候，你需要注意：当你加载一个 XIB 时，它的所有内容都会被加载，如果这个 XIB 里面有个 View 你不会马上就用到，你其实就是在浪费宝贵的内存。而加载 StoryBoard 时并不会把所有的 ViewController 都加载，只会按需加载。


###不要阻塞主线程
基本上 UIKit 会把它所有的工作都放在主线程执行，比如：绘制界面、管理手势、响应输入等等。当你把所有代码逻辑都放在主线程时，有可能因为耗时太长卡住主线程造成程序无法响应、流畅性太差等问题。造成这种问题的大多数场景是因为你的程序把 I/O 操作放在了主线程，比如从硬盘或者网络读写数据等等。

你可以通过异步的方式来进行这些操作，把他们放在别的线程中处理。比如处理网络请求时，你可以使用 NSURLConnection 的异步调用 API：

	+ (void)sendAsynchronousRequest:(NSURLRequest *)request queue:(NSOperationQueue *)queue completionHandler:(void (^)(NSURLResponse*, NSData*, NSError*))handler;
	
或者使用第三方的类库，比如 [AFNetworking](https://github.com/AFNetworking/AFNetworking)。

当你做一些耗时比较长的操作时，你可以使用 GCD、NSOperation、NSOperationQueue。比如 GCD 的常见使用方式：

	dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
	    // switch to another thread and perform your expensive operation
	 
	    dispatch_async(dispatch_get_main_queue(), ^{
	        // switch back to the main thread to update your UI
	 
	    });
	});

关于 GCD 更多的知识，你可以看看这篇文章：[GCD](http://www.samirchen.com/ios-gcd/)。


###图片尺寸匹配 UIImageView
当你从 App bundle 中加载图片到 UIImageView 中显示时，最好确保图片的尺寸能够和 UIImageView 的尺寸想匹配（当然，需要考虑 @2x @3x 的情况），否则会使得 UIImageView 在显示图片时需要做拉伸，这样会影响性能，尤其是在一个  UIScrollView 的容器里。

有时候，你的图片是从网络加载的，这时候你并不能控制图片的尺寸，不过你可以在图片下载下来后去手动 scale 一下它，当然，最好是在一个后台线程做这件事，然后在 UIImageView 中使用 resize 后的图片。


###选择合适的容器
我们经常需要用到容器来转载多个对象，我们通常用到的包括：NSArray、NSDictionary、NSSet，它们的特性如下：

- Array：数组。有序的，通过 index 查找很快，通过 value 查找很慢，插入和删除较慢。
- Dictionary：字典。存储键值对，通过键查找很快。
- Set：集合。无序的，通过 value 查找很快，插入和删除较快。

根据以上特性，在编程中需要选择适合的容器。更多内容请看：[Collections Programming Topics](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Collections/Collections.html)


###启用 GZIP 数据压缩
现在越来越多的应用需要跟服务器进行数据交互，当交互的数据量较大时，网络传输的时延就会较长，通过启动数据压缩功能，尤其是对于文本信息，可以降低网络传输的数据量，从而减短网络交互的时间。

一个好消息是当你使用 NSURLConnection 或者基于此的一些网络交互类库（比如 AFNetworking）时 iOS 已经默认支持 GZIP 压缩。并且，很多服务器已经支持发送压缩数据。

通过在服务器和客户端程序中启用对网络交互数据的压缩，是一条提高应用程序性能的途径。




##中级技巧
在上面的内容里我们介绍了一些显而易见的优化程序性能的途径，但是有时候，有些优化程序性能的方案并不是那么明显，这些方案是否适用取决于你的代码情况。但是，如果在正确的场景下，这些方案能起到很显著的作用。


###View 的复用和懒加载机制
当你的程序中需要展示很多的 View 的时候，这就意味着需要更多的 CPU 处理时间和内存空间，这个情况对程序性能的影响在你使用 UIScrollView 来装载和呈现界面时会变得尤为显著。

处理这种情况的一种方案就是向 UITableView 和 UICollectionView 学习，不要一次性把所有的 subviews 都创建出来，而是在你需要他们的时候创建，并且用复用机制去复用他们。这样减少了内存分配的开销，节省了内存空间。

「懒加载机制」就是把创建对象的时机延后到不得不需要它们的时候。这个机制常常用在对一个类的属性的初始化上，比如：

	- (UITableView *)myTableView {
	    if (!_myTableView) {
	        CGRect viewBounds = self.view.bounds;
	        	        
	        _myTableView = [[UITableView alloc] initWithFrame:viewBounds style:UITableViewStylePlain];
	        _myTableView.showsHorizontalScrollIndicator = NO;
	        _myTableView.showsVerticalScrollIndicator = NO;
	        _myTableView.backgroundColor = [UIColor whiteColor];
	        [_myTableView setSeparatorStyle:UITableViewCellSeparatorStyleNone];
	        
	        _myTableView.dataSource = self;
	        _myTableView.delegate = self;
	    }
	    
	    return _myTableView;
	}

只有当我们第一次用到 self.myTableView 的时候采取初始化和创建它。

但是，存在这样一种场景：你点击一个按钮的时候，你需要显示一个 View，这时候你有两种实现方案：

- 1）在当前界面第一次加载的时候就创建出这个 View，只是把它隐藏起来，当你需要它的时候，只用显示它就行了。
- 2）使用「懒加载机制」，在你需要这个 View 的时候才创建它，并展示它。

这两种方案都各有利弊。采用方案一，你在不需要这个 View 的时候显然白白地占用了更多的内存，但是当你点击按钮展示它的时候，你的程序能响应地相对较快，因为你只需要改变它的 hidden 属性。采用方案二，那么你得到的效果相反，你更准确的使用了内存，但是如果对这个 View 的初始化和创建比较耗时，那么响应性相对就没那么好了。

所以当你考虑使用何种方案时，你需要根据现实的情况来参考，去权衡到底哪个因素才是影响性能的瓶颈，然后再做出选择。


###缓存
在开发我们的程序时，一个很重要的经验法则就是：对那些更新频度低，访问频度高的内容做缓存。

有哪些东西使我们可以缓存的呢？比如下面这些：

- 服务器的响应信息（response）。
- 图片。
- 计算值。比如：UITableView 的 row heights。

NSURLConnection 可以根据 HTTP 头部的设置来决定把资源内容缓存在磁盘或者内存，你甚至可以设置让它只加载缓存里的内容：

	+ (NSMutableURLRequest *)imageRequestWithURL:(NSURL *)url {
	    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
	 
	    request.cachePolicy = NSURLRequestReturnCacheDataElseLoad; // this will make sure the request always returns the cached image
	    request.HTTPShouldHandleCookies = NO;
	    request.HTTPShouldUsePipelining = YES;
	    [request addValue:@"image/*" forHTTPHeaderField:@"Accept"];
	 
	    return request;
	}

关于 HTTP 缓存的更多内容可以关注 NSURLCache。关于缓存其他非 HTTP 请求的内容，可以关注 NSCache。对于图片缓存，可以关注一个第三方库 [SDWebImage](https://github.com/rs/SDWebImage)。


###关于图形绘制
当我们为一个 UIButton 设置背景图片时，对于这个背景图片的处理，我们有很多种方案，你可以使用全尺寸图片直接设置，还可以用 resizable images，或者使用 CALayer、CoreGraphics 甚至 OpenGL 来绘制。

当然，不同的方案的编码复杂度不一样，性能也不一样。关于图形绘制的不同方案的性能问题，可以看看：[Designing for iOS: Graphics Performance](https://robots.thoughtbot.com/designing-for-ios-graphics-performance)

简而言之，使用 pre-rendered 的图片会更快，因为这样就不需要在程序中去创建一个图像，并在上面绘制各种形状了（Offscreen Rendering，离屏渲染）。但是缺点是你必须把这些图片资源打包到代码包，从而需要增加程序包的体积。这就是为什么 resizable images 是一个很棒的选择：不需要全尺寸图，让 iOS 为你绘制图片中那些可以拉伸的部分，从而减小了图片体积；并且你不需要为不同大小的控件准备不同尺寸的图片。比如两个按钮的大小不一样，但是他们的背景图样式是一样的，你只需要准备一个对应样式的 resizable image，然后在设置这两个按钮的背景图的时候分别做拉伸就可以了。

但是一味的使用使用预置的图片也会有一些缺点，比如你做一些简单的动画的时候各个帧都用图片叠加，这样就可能要使用大量图片。

总之，你需要去在图形绘制的性能和应用程序包的大小上做权衡，找到最合适的性能优化方案。


###处理 Memory Warnings
关于内存警告，苹果的官方文档是这样说的：

>If your app receives this warning, it must free up as much memory as possible. The best way to do this is to remove strong references to caches, image objects, and other data objects that can be recreated later.


我们可以通过这些方式来获得内存警告：

- 在 AppDelegate 中实现 `- [AppDelegate applicationDidReceiveMemoryWarning:]` 代理方法。
- 在 UIViewController 中重载 `didReceiveMemoryWarning` 方法。
- 监听 `UIApplicationDidReceiveMemoryWarningNotification` 通知。 

当通过这些方式监听到内存警告时，你需要马上释放掉不需要的内存从而避免程序被系统杀掉。

比如，在一个 UIViewController 中，你可以清除那些当前不显示的 View，同时可以清除这些 View 对应的内存中的数据，而有图片缓存机制的话也可以在这时候释放掉不显示在屏幕上的图片资源。

但是需要注意的是，你这时清除的数据，必须是可以在重新获取到的，否则可能因为必要数据为空，造成程序出错。在开发的时候，可以使用 iOS Simulator 的 `Simulate memory warning` 的功能来测试你处理内存警告的代码。


###复用高开销的对象
在 Objective-C 中有些对象的初始化过程很缓慢，比如：`NSDateFormatter` 和 `NSCalendar`，但是有些时候，你也不得不使用它们。为了这样的高开销的对象成为影响程序性能的重要因素，我们可以复用它们。

比如，我们在一个类里添加一个 NSDateFormatter 的对象，并使用懒加载机制来使用它，整个类只用到一个这样的对象，并只初始化一次：

	// in your .h or inside a class extension
	@property (nonatomic, strong) NSDateFormatter *dateFormatter;
	 
	// inside the implementation (.m)
	// When you need, just use self.dateFormatter
	- (NSDateFormatter *)dateFormatter {
	    if (! _dateFormatter) {
	        _dateFormatter = [[NSDateFormatter alloc] init];
            [_dateFormatter setDateFormat:@"yyyy-MM-dd a HH:mm:ss EEEE"];
	    }
	    return _dateFormatter;
	}

但是上面的代码在多线程环境下会有问题，所以我们可以改进如下：

	// no property is required anymore. The following code goes inside the implementation (.m)
	- (NSDateFormatter *)dateFormatter {
	    static NSDateFormatter *dateFormatter;
	    static dispatch_once_t onceToken;
	    dispatch_once(&onceToken, ^{
	        dateFormatter = [[NSDateFormatter alloc] init];
            [dateFormatter setDateFormat:@"yyyy-MM-dd a HH:mm:ss EEEE"];
	    });
	    return dateFormatter;
	}

这样就线程安全了。（关于多线程 GCD 的知识，可以看看这篇文章：[GCD](http://www.samirchen.com/ios-gcd/)）

需要注意的是：设置 NSDateFormatter 的 date format 跟创建一个新的 NSDateFormatter 对象一样慢，因此当你的程序中要用到多种格式的 date format，而每种又会用到多次的时候，你可以尝试为每种 date format 创建一个可复用的 NSDateFormatter 对象来提供程序的性能。

###在游戏中使用 Sprite Sheet
如果你是游戏开发者，使用 Sprite Sheet 可以帮助你比标准的绘图方法更快地绘制场景，甚至占用更少的内存。

当然这里有些地方也可以参考上面已经提到过的性能优化方案，比如你的游戏有很多 Sprites 时，你可以参考 UITableViewCell 的复用机制，而不是每次都创建它们。


###避免倒腾数据
在我们开发应用时，经常会遇到要从服务器获取 JSON 或者 XML 数据来处理的情况，这时我们通常都需要解析这些数据，一般会解析为 NSArray、NSDictionary 的对象。但是在我们实际的开发中，我们通常会为在界面上展示的那些数据定义一些数据结构（ViewModel）。这时候问题就来了，我们需要把解析出来的 NSArray、NSDictionary 对象再倒腾成我们定义的那些数据结构，从程序性能的角度来考虑，这是一项开销较大的操作。

为了避免这样的情况成为影响程序性能的瓶颈，在设计客户端应用程序对应的数据结构时就需要做更细致的考虑，尽量用 NSArray 去承接 NSArray，用 NSDictionary 去承接 NSDictionary，避免倒腾数据造成开销。


###选择正确的数据格式
我们的 iOS 应用程序与服务器进行交互时，通常采用的数据格式就是 JSON 和 XML 两种。那么在选择哪一种时，需要考虑到它们的优缺点。

JSON 文件的优点是：

- 能够更快的被解析。
- 在承载相同的数据时，通常体积比 XML 更小，这意味着传输的数据量更小。

缺点是：

- 需要整个 JSON 数据全部加载完成后才能开始解析。

而 XML 文件的优缺点则刚好反过来。 XML 的一个优点就是它可以使用 [SAX](https://en.wikipedia.org/wiki/Simple_API_for_XML) 来解析数据，从而可以边加载边解析，不用等所有数据都读取完成了才解析。这样在处理很大的数据集的时提高性能和降低内存消耗。

所以，你需要根据具体的应用场景来权衡使用何种数据格式。


###合理的设置背景图片
我们通常有两种方式来设置一个 View 的背景图片：

- 通过 `- [UIColor colorWithPatternImage:]` 方法来设置 View 的 background color。
- 通过给 View 添加一个 UIImageView 来设置其背景图片。

当你有一个全尺寸图片作为背景图时，你最好用 UIImageView 来，因为 `- [UIColor colorWithPatternImage:]` 是用来可重复填充的小的样式图片。这时对于全尺寸的图片，用 UIImageView 会节省大量的内存。
	
	// You could also achieve the same result in Interface Builder
	UIImageView *backgroundView = [[UIImageView alloc] initWithImage:[UIImage imageNamed:@"background"]];
	[self.view addSubview:backgroundView];

但是，当你计划采用一个小块的模板样式图片，就像贴瓷砖那样来重复填充整个背景时，你应该用 `- [UIColor colorWithPatternImage:]` 这个方法，因为这时它能够绘制的更快，并且不会用到太多的内存。

	self.view.backgroundColor = [UIColor colorWithPatternImage:[UIImage imageNamed:@"backgroundPattern"]];



###优化 WebView
UIWebView 在我们的应用程序中非常有用，它可以便捷的展示 Web 的内容，甚至做到你用标准的 UIKit 控件较难做到的视觉效果。但是，你应该注意到你在应用程序里使用的 UIWebView 组件不会比苹果的 Safari 更快。这是首先于 Webkit 的 Nitro Engine 引擎。所以，为了得到更好的性能，你需要优化你的网页内容。

优化第一步就是避免过量使用 Javascript，例如避免使用较大的 Javascript 框架，比如 jQuery。一般使用原生的 Javascript 而不是依赖于 Javascript 框架可以获得更好的性能。

优化第二步，如果可能的话，可以异步加载那些不影响页面行为的 Javascript 脚本，比如一些数据统计脚本。

优化第三步，总是关注你在页面中所使用的图片，根据具体的场景来显示正确尺寸的图片，同时也可以使用上面提到的「使用 Sprites Sheets」的方案来在某些地方减少内存消耗和提高速度。

###减少离屏渲染
什么是「离屏渲染」？离屏渲染，即 Off-Screen Rendering。与之相对的是 On-Screen Rendering，即在当前屏幕渲染，意思是渲染操作是用于在当前屏幕显示的缓冲区进行。那么离屏渲染则是指图层在被显示之前是在当前屏幕缓冲区以外开辟的一个缓冲区进行渲染操作。

离屏渲染需要多次切换上下文环境：先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上又需要将上下文环境从离屏切换到当前屏幕，而上下文环境的切换是一项高开销的动作。

通常图层的以下属性将会触发离屏渲染：

- 阴影（UIView.layer.shadowOffset/shadowRadius/...）
- 圆角（当 UIView.layer.cornerRadius 和 UIView.layer.maskToBounds 一起使用时）
- 图层蒙板


在 iOS 开发中要给一个 View 添加阴影效果，有很简单快捷的做法：
 
	UIImageView *imageView = [[UIImageView alloc] initWithFrame:...];
	 
	// Setup the shadow ...
	imageView.layer.shadowOffset = CGSizeMake(5.0f, 5.0f);
	imageView.layer.shadowRadius = 5.0f;
	imageView.layer.shadowOpacity = 0.6;
	

但是上面这样的做法有一个坏处是：将触发 Core Animation 做离屏渲染造成开销。

那要做到阴影图层效果，又想减少离屏渲染、提高性能的话要怎么做呢？一个好的建议是：设置 ShadowPath 属性。

	UIImageView *imageView = [[UIImageView alloc] initFrame:...];
	 
	// Setup the shadow ...
	imageView.layer.shadowPath = [[UIBezierPath bezierPathWithRect:CGRectMake(imageView.bounds.origin.x+5, imageView.bounds.origin.y+5, imageView.bounds.size.width, imageView.bounds.size.height)] CGPath];
	imageView.layer.shadowOpacity = 0.6;

如果图层是一个简单几何图形如矩形或者圆角矩形（假设不包含任何透明部分或者子图层），通过设置 ShadowPath 属性来创建出一个对应形状的阴影路径就比较容易，而且 Core Animation 绘制这个阴影也相当简单，不会触发离屏渲染，这对性能来说很有帮助。如果你的图层是一个更复杂的图形，生成正确的阴影路径可能就比较难了，这样子的话你可以考虑用绘图软件预先生成一个阴影背景图。


###光栅化
CALayer 有一个属性是 `shouldRasterize` 通过设置这个属性为 YES 可以将图层绘制到一个屏幕外的图像，然后这个图像将会被缓存起来并绘制到实际图层的 contents 和子图层，如果很很多的子图层或者有复杂的效果应用，这样做就会比重绘所有事务的所有帧来更加高效。但是光栅化原始图像需要时间，而且会消耗额外的内存。这是需要根据实际场景权衡的地方。

当我们使用得当时，光栅化可以提供很大的性能优势，但是一定要避免在内容不断变动的图层上使用，否则它缓存方面的好处就会消失，而且会让性能变的更糟。

为了检测你是否正确地使用了光栅化方式，可以用 Instrument 的 Core Animation Template 查看一下 `Color Hits Green and Misses Red` 项目，看看是否已光栅化图像被频繁地刷新（这样就说明图层并不是光栅化的好选择，或则你无意间触发了不必要的改变导致了重绘行为）。

如果你最后设置了 shouldRasterize 为 YES，那也要记住设置 rasterizationScale 为合适的值。在我们使用 UITableView 和 UICollectionView 时经常会遇到各个 Cell 的样式是一样的，这时候我们可以使用这个属性提高性能：

	cell.layer.shouldRasterize = YES;
	cell.layer.rasterizationScale = [[UIScreen mainScreen] scale];

但是，如果你的 Cell 是样式不一样，比如高度不定，排版多变，那就要慎重了。


###优化 UITableView
UITableView 是我们最常用来展示数据的控件之一，并且通常需要 UITableView 在承载较多内容的同时保证交互的流畅性，对 UITableView 的性能优化是我们开发应用程序必备的技巧之一。

在前文「使用复用机制」一节，已经提到了 UITableView 的复用机制。现在就来看看 UITableView 在复用时最主要的两个回调方法：`- [UITableView tableView:cellForRowAtIndexPath:]` 和 `- [UITableView tableView:heightForRowAtIndexPath:]`。UITableView 是继承自 UIScrollView，所以在渲染的过程中它会先确定它的 contentSize 及每个 Cell 的位置，然后才会把复用的 Cell 放置到对应的位置。比如现在一共有 50 个 Cell，当前屏幕上显示 5 个。那么在第一次创建或 reloadData 的时候， UITableView 会先调用 50 次 `- [UITableView tableView:heightForRowAtIndexPath:]` 确定 contentSize 及每个 Cell 的位置，然后再调用 5 次 `- [UITableView tableView:cellForRowAtIndexPath:]` 来渲染当前屏幕的 Cell。在滑动屏幕的时候，每当一个 Cell 进入屏幕时，都需要调用一次 `- [UITableView tableView:cellForRowAtIndexPath:]` 和 `- [UITableView tableView:heightForRowAtIndexPath:]` 方法。

了解了 UITableView 的复用机制以及相关回调方法的调用次序，这里就对 UITableView 的性能优化方案做一个总结：

- 通过正确的设置 reuseIdentifier 来重用 Cell。
- 尽量减少不必要的透明 View。
- 尽量避免渐变效果、图片拉伸和离屏渲染。
- 当不同的行的高度不一样时，尽量缓存它们的高度值。
- 如果 Cell 展示的内容来自网络，确保用异步加载的方式来获取数据，并且缓存服务器的 response。
- 使用 shadowPath 来设置阴影效果。
- 尽量减少 subview 的数量，对于 subview 较多并且样式多变的 Cell，可以考虑用异步绘制或重写 drawRect。
- 尽量优化 `- [UITableView tableView:cellForRowAtIndexPath:]` 方法中的处理逻辑，如果确实要做一些处理，可以考虑做一次，缓存结果。
- 选择合适的数据结构来承载数据，不同的数据结构对不同操作的开销是存在差异的。
- 对于 rowHeight、sectionFooterHeight、sectionHeaderHeight 尽量使用常量。


###选择合适的数据存储方式
在 iOS 中可以用来进行数据持有化的方案包括：

- NSUserDefaults。只适合用来存小数据。
- XML、JSON、Plist 等文件。JSON 和 XML 文件的差异在「选择正确的数据格式」已经说过了。
- 使用 NSCoding 来存档。NSCoding 同样是对文件进行读写，所以它也会面临必须加载整个文件才能继续的问题。
- 使用 SQLite 数据库。可以配合 [FMDB](https://github.com/ccgus/fmdb) 使用。数据的相对文件来说还是好处很多的，比如可以按需取数据、不用暴力查找等等。
- 使用 CoreData。也是数据库技术，跟 SQLite 的性能差异比较小。但是 CoreData 是一个对象图谱模型，显得更面向对象；SQLite 就是常规的 DBMS。




##高级技巧
下面介绍几个高级技巧。

###减少应用启动时间
快速启动应用对于用户来说可以留下很好的印象。尤其是第一次使用时。

保证应用快速启动的指导原则：

- 尽量将启动过程中的处理分拆成各个异步处理流，比如：网络请求、数据库访问、数据解析等等。
- 避免臃肿的 XIB 文件，因为它们会在你的主线程中进行加载。重申：Storyboard 没这个问题，放心使用。

注意：在测试程序启动性能的时候，最好用与 Xcode 断开连接的设备进行测试。因为 watchdog 在使用 Xcode 进行调试的时候是不会启动的。

###使用 Autorelease Pool
NSAutoreleasePool 是用来管理一个自动释放内存池的机制。在我们的应用程序中通常都是 UIKit 隐式的自动使用 Autorelease Pool，但是有时候我们也可以显式的来用它。

比如当你需要在代码中创建许多临时对象时，你会发现内存消耗激增直到这些对象被释放，一个问题是这些内存只会到 UIKit 销毁了它对应的 Autorelease Pool 后才会被释放，这就意味着这些内存不必要地会空占一些时间。这时候就是我们显式的使用 Autorelease Pool 的时候了，一个示例如下：

	NSArray *urls = <# An array of file URLs #>;
	for (NSURL *url in urls) {
	    @autoreleasepool {
	        NSError *error;
	        NSString *fileContents = [NSString stringWithContentsOfURL:url
	                                         encoding:NSUTF8StringEncoding error:&error];
	        /* Process the string, creating and autoreleasing more objects. */
	    }
	}

上面的代码在每一轮迭代中都会释放掉临时对象，从而缓解内存压力，提高性能。

关于 Autorelease Pool 你还可以看看：[Using Autorelease Pool Blocks](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html) 和 [iOS 中的 AutoreleasePool](http://www.samirchen.com/ios-autorelease-pool/)。

###imageNamed 和 imageWithContentsOfFile
在 iOS 应用中加载图片通常有 `- [UIImage imageNamed:]` 和 `-[UIImage imageWithContentsOfFile:]` 两种方式。它们的不同在于前者会对图片进行缓存，而后者只是简单的从文件加载文件。

	UIImage *img = [UIImage imageNamed:@"myImage"]; // caching
	// or
	UIImage *img = [UIImage imageWithContentsOfFile:@"myImage"]; // no caching

在整个程序运行的过程中，当你需要加载一张较大的图片，并且只会使用它一次，那么你就没必要缓存这个图片，这时你可以使用 `-[UIImage imageWithContentsOfFile:]`，这样系统也不会浪费内存来做缓存了。当然，如果你会多次使用到一张图时，用 `- [UIImage imageNamed:]` 就会高效很多，因为这样就不用每次都从硬盘上加载图片了。


###避免使用 NSDateFormatter
在前文中，我们已经讲到了通过复用或者单例来提高 NSDateFormatter 这个高开销对象的使用效率。但是如果你要追求更快的速度，你可以直接使用 C 语言替代 NSDateFormatter 来解析 date，你可以看看这篇文章：[link](http://blog.soff.es/how-to-drastically-improve-your-app-with-an-afternoon-and-instruments)，其中展示了解析 ISO-8601 date string 的代码，你可以根据你的需求改写。完成的代码见：[SSToolkit/NSDate+SSToolkitAdditions.m](https://github.com/samsoffes/sstoolkit/blob/master/SSToolkit/NSDate%2BSSToolkitAdditions.m)。

当然，如果你能够控制你接受到的 date 的参数的格式，你一定要尽量选择 `Unix timestamps` 格式，这样你可以使用：

	- (NSDate*)dateFromUnixTimestamp:(NSTimeInterval)timestamp {
	    return [NSDate dateWithTimeIntervalSince1970:timestamp];
	}

这样你可以轻松的将时间戳转化为 NSDate 对象，并且效率甚至高于上面提到的 C 函数。

需要注意的是，很多 web API 返回的时间戳是以**毫秒**为单位的，因为这更利于 Javascript 去处理，但是上面代码用到的方法中 NSTimeInterval 的单位是**秒**，所以当你传参的时候，记得先除以 1000。


###IMP Caching
在 Objective-C 的消息分发过程中，所有 `[receiver message:…]` 形式的方法调用最终都会被编译器转化为 `obj_msgSend(recevier, @selector(message), …)` 的形式调用。在运行时，Runtime 会去根据 selector 到对应方法列表中查找相应的 IMP 来调用，这是一个动态绑定的过程。为了加速消息的处理，Runtime 系统会缓存使用过的 selector 对应的 IMP 以便后面直接调用，这就是 IMP Caching。通过 IMP Caching 的方式，Rumtime 能够跳过 obj_msgSend 的过程直接调用方法的实现，从而提高方法调用效率。



下面看段示例代码：

	#define LOOP 1000000
	#define START { clock_t start, end; start = clock();
	#define END end = clock(); printf("Cost: %f ms\n", (double)(end - start) / CLOCKS_PER_SEC * 1000); }

	- (NSDateFormatter *)dateFormatter:(NSString *)format {
	     NSDateFormatter *dateFormatter = [[NSDateFormatter alloc] init];
	    [dateFormatter setDateFormat:format];
	    
	    return dateFormatter;
	}


	- (void)testIMPCaching {
	    [self normalCall];
	    
	    [self impCachingCall];
	}

	- (void)normalCall {
	    START
	   
	    for (int32_t i = 0; i < LOOP; i++) {
	        NSDateFormatter *d =[self dateFormatter:@"yyyy-MM-dd a HH:mm:ss EEEE"];
	        d = nil;
	    }

	    END
	    // Print: Cost: 1328.845000 ms
	}

	- (void)impCachingCall {
	    START
	    
	    SEL sel = @selector(dateFormatter:);
	    NSDateFormatter *(*imp)(id, SEL, id) = (NSDateFormatter *(*)(id, SEL, id)) [self methodForSelector:sel];
	    
	    for (int32_t i = 0; i < LOOP; i++) {
	        NSDateFormatter *d = imp(self, sel, @"yyyy-MM-dd a HH:mm:ss EEEE");
	        d = nil;
	    }
	    
	    END
	    // Print: Cost: 1130.200000 ms
	}


代码打印结果如下：

	Cost: 1328.845000 ms
	Cost: 1130.200000 ms

可见相差并不太大，在 `impCachingCall` 中是直接手动做 IMP Caching 来跳过 obj_msgSend 调用方法的实现。`normalCall` 则是在 Runtime 通过系统自己的 IMP Caching 机制来运行。通常我们不需要做 IMP Caching，但是如果有时候哪怕一点点的速度提升也是你需要的，你可以考虑考虑这点。

此外关于 Objective 运行时相关的知识，你可以看看这篇：[Objective-C 的 Runtime][4]


参考：

- [25 iOS App Performance Tips & Tricks][3]





[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://samirchen.com/ios-performance-optimization
[3]: http://www.raywenderlich.com/31166/25-ios-app-performance-tips-tricks
[4]: http://www.samirchen.com/objective-c-runtime