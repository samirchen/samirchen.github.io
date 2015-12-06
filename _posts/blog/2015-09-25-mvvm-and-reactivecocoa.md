---
layout: post
title: MVVM 模式及 ReactiveCocoa
description: MVVM 和 ReactiveCocoa 已经被很多开发者熟悉，网上也有很多相关文章，我这里也简单聊一聊，主要算是自己的一个笔记。
category: blog
tag: iOS, Objective-C, MVVM, ReactiveCocoa
---


## 为什么要 MVVM

MVVM 的最大作用是为了解决 Massive View Controller 的问题，为 Cocoa MVC 模式下臃肿的 ViewController 瘦身。那 Cocoa MVC 模式下的 ViewController 为什么会「胖」呢？主要是因为 ViewController 自带一个 View，并且需要管理基于这个 View 之上的整个 View Hierarchy，控制它们的展示逻辑以及它们和 Model 层的数据交互，所有这些代码都塞在一个 ViewController 里，一旦页面内容丰富一点，难保不「胖」。而 MVVM 则是为这一问题瘦身的方案之一。




## MVVM 怎么解决问题

那 MVVM 是怎么做的呢？MVVM 是 Model-View-ViewModel 的缩写，其中最重要的概念就是 ViewModel。在 MVVM 中，将 MVC 的结构方式做了改变：

- 将 View 和 ViewController 看做同类。
- 在 View(ViewController) 和 Model 层之间加入一层 ViewModel 层。

来自 [Introduction to MVVM][3] 的两张图清晰的反应了 MVC 和 MVVM 的结构差别：

![image](../../images/mvvm-and-reactivecocoa/mvc.png)

![image](../../images/mvvm-and-reactivecocoa/mvvm.png)

在 MVVM 的结构下，View(ViewController) 不再直接与 Model 打交道，而是与新加入的 ViewModel 打交道，原来塞在 View(ViewController) 中的展示逻辑则转移到 ViewModel 中去，从而为 View(ViewController) 瘦身。





## MVVM 带来的好处和新问题

通过 MVVM 这一结构，首先解决了 MVC 中 ViewController 过于臃肿的问题，View(ViewController) 的数据处理逻辑移至 ViewModel 中，从而专注于界面展示，不再需要处理数据逻辑，变得轻盈，从而也更加便于测试。并且这种新的结构与 MVC 是兼容的。

但是，这样一来也带来了新的问题：由于隔了一层 ViewModel，当 Model 发生变化时，就没那么方便直接通知到 View(ViewController) 了。但是这个问题也有很多解决方案，比如用 KVO、Notification、Delegate、Block 回调等等，但这里我们要说的是 **ReactiveCocoa**。



## MVVM 搭配 ReactiveCocoa

ReactiveCocoa 是一个具有函数式编程和响应式编程特性的开发框架，它为我们提供了统一的消息传递机制。ReactiveCocoa 中最重要的概念是 **Signal(信号)**，即整个数据流中流动的信号，Signal 需要 **Subcriber(订阅者)** 来订阅它，当信号源对应的数据发生变化，它发出信号，然后信号被发送给对应的订阅者，订阅者获取到信号并获取随之而来的数据。当然，ReactiveCocoa 实现了很多更复杂的机制和能力，这里就不一一介绍了。

ReactiveCocoa 的信号机制正好可以解决我们上文中所说的 MVVM 带来的问题：Model 和 View(ViewController) 之间的数据绑定。




## MVVM 实践

我结合对 MVVM 的理解写了一个 Demo，可以到这里下载：[https://github.com/samirchen/MVVMDemo][4]。里面展示了 Model、ViewModel、ViewController、View 各个层次的代码应该如何写。

### Model

如果你想要在你的项目中使用 Model 层的话，这一层的定义通常是这样的：以面向对象的思想抽象出来的实体类的封装。一个 Model 通常映射着数据的一张表。同时，这一层还提供与每个 Model 对应的服务接口，比如：从服务器请求 Model 相关的数据后，将数据处理成 Model 实例返回出去。

比如 Demo 中的 `CXMovie` 这个 Model 实体的代码如下：

	// CXMovie.h
	#import <Foundation/Foundation.h>

	@interface CXMovie : NSObject

	@property (nonatomic, assign) int64_t rowid;
	@property (nonatomic, strong) NSString *posterImageURL;
	@property (nonatomic, strong) NSString *name;
	@property (nonatomic, strong) NSDate *releaseTime;

	@end

`CXMovieService` 是与 `CXMovie` 相关的数据服务，代码如下：

	// CXMovieService.h
	@interface CXMovieService : NSObject

	+ (void)requestMovieDataWithParameters:(id)parameters start:(void (^)(void))start success:(void (^)(NSArray *movieList, NSString *successMessage))success failure:(void (^)(NSError *error, NSString *failureMessage))failure;
	+ (RACSignal *)signalWhenRequstMovieDataWithParameters:(id)parameters;

	@end

可以看到这里，我们提供了两种类型的接口，一种是以 Block 回调的方式供使用方调用，一种则是用了 ReactiveCocoa 的信号。使用信号的流程上面已经介绍过了，在这里大致就是：使用方调用请求数据的接口去得到对应的信号实例，并订阅它，当数据请求成功或失败时，信号实例会发送消息给订阅者，订阅者收到消息后做后续的处理。


### ViewModel

一方面 ViewModel 对接 Model 层的作为其数据源，Model 层的实例以及相关数据服务通常的使用方都是 ViewModel；另一方面 ViewModel 提供展示数据的接口来满足 View 或 ViewController 展示数据的需要。


比如 Demo 中的 CXMovieCellViewModel 的代码如下：

	// CXMovieCellViewModel.h
	#import <Foundation/Foundation.h>
	#import <UIKit/UIKit.h>
	#import "CXMovie.h"

	@interface CXMovieCellViewModel : NSObject

	// 对接 Model 层的数据源。
	@property (nonatomic, strong) CXMovie *movie;

	// 对接 View(ViewController) 层的数据接口，满足展示需求。
	- (NSURL *)moviePosterURL;
	- (NSString *)movieNameText;
	- (NSString *)movieReleaseTimeText;

	@end

`CXMovieCellViewModel` 是一个 ViewModel，其中包含了一个 Model 类的属性 `CXMovie *movie`，这个其实就是它的数据源，`CXMovieCellViewModel` 基于 `movie` 这个数据源的数据进行一些处理后，再以 `- (NSString *)movieNameText;` 等接口的方式提供给 View(ViewController) 层使用。


### View(ViewController)

View(ViewController) 只做展示页面的事情，所有数据处理逻辑都交给 ViewModel 了，在 View(ViewController) 中则根据展示需要向 ViewModel 要数据即可。

比如：

	- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
	    return [self.viewModel movieCellCount];
	}

	- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
	    CXMovieTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:CXMovieCellIdentifier forIndexPath:indexPath];
	    
	    CXMovieCellViewModel *cellViewModel = [self.viewModel movieCellViewModelAtIndexPath:indexPath];
	    cell.viewModel = cellViewModel;
	    
	    return cell;
	}

通常在一个 ViewController 里使用的一些模块化的 View 可以拆出来单独封装，这时候我们在 ViewModel 层可以对每个子 View 做一个对应的 ViewModel。比如一个 ViewController 中的包含一个 TableView，TableView 的每个 Cell 是自定义的。这时候 ViewController 和 Cell 各对应一个 ViewModel。在我们的 Demo 中我们就有体现这一点，具体的代码请去下载查阅。






[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://samirchen.com/mvvm-and-reactivecocoa/
[3]: https://www.objc.io/issues/13-architecture/mvvm/
[4]: https://github.com/samirchen/MVVMDemo




