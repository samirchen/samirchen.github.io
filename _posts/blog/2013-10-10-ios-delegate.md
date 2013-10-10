---
layout: post
title: iOS开发中的Delegate模式使用示例
description: 简要介绍了一下iOS开发中的Delegate模式使用的简单示例。
category: blog
---

##1、Delegate模式介绍：
delegation，委托模式（另外有个常用的proxy模式，二者的区别是代理模式一般要更严格，实现相同的接口，委托只是引用被委托对象），是简单的强大的模式，可让一个对象扮演另外对象的行为。委托对象保持到另外对象的引用，并在适当的时候发消息给另外对象。委托对象可以在发送消息的时候做一些额外的事情。

在cocoa框架中的委托模式，委托对象往往是框架中的对象，被委托对象是自定义的controller对象。委托对象保持一个到被委托对象的弱引用。

##2、示例：

2.1）代码：`CPPropOperationView.h`

在这里定义了 `CPPropOperationViewDelegate`，它有两个 `required` 的方法 `refreshPropOperationView:` 和 `closePropOperationView:` ，所有实现这个 `CPPropOperationViewDelegate` 的类需要去实现这两个方法。

接着，声明了 `CPPropOperationView` 这个类的一些属性，其中包括 `id delegate`。

	#import <UIKit/UIKit.h>
	 
	#pragma mark - delegate
	@protocol CPPropOperationViewDelegate <NSObject>
	@required
	-(void) refreshPropOperationView:(id)sender;
	-(void) closePropOperationView:(id)sender; 
	@end
	 
	#pragma mark - class
	@interface CPPropOperationView : UIView {
	}
	@property (nonatomic, assign) id<CPPropOperationViewDelegate> delegate;
	@property (nonatomic, retain) UIImageView* backgroundImageView;
	@property (nonatomic, retain) UIButton* refreshBtn;
	@property (nonatomic, retain) UIButton* closeBtn;
	@end

2.2）代码：`CPPropOperationView.m`

这里是对 `CPPropOperationView` 页面元素的一些实现代码。可以看到，我们在给 `CPPropOperationView` 的两个按钮 `refreshBtn` 和 `closeBtn` 绑定方法的时候，`addTarget:` 的对象是 `delegate`，这说明我们将用代理调用这些按钮对应的方法。

	#import "CPPropOperationView.h"
	 
	@implementation CPPropOperationView
	@synthesize delegate;
	@synthesize backgroundImageView = _backgroundImageView;
	@synthesize refreshBtn = _refreshBtn;
	@synthesize closeBtn = _closeBtn;
	 
	#pragma mark - View Life Cycle
	- (id)initWithFrame:(CGRect)frame {
	    self = [super initWithFrame:frame];
	    if (self) {
	        // Initialization code       
	        
	        // background image.
	        self.backgroundImageView = [[[UIImageView alloc] initWithImage:[UIImage imageNamed:@"bg.png"]] autorelease];
	        [self.backgroundImageView setFrame:CGRectMake(0, 0, frame.size.width, frame.size.height)];
	        [self addSubview:self.backgroundImageView];
	         
	        // Refresh button. Bind to delegate's refreshPropOperationView:.
	        self.refreshBtn = [UIButton buttonWithType:UIButtonTypeCustom];
	        [self.refreshBtn setFrame:CGRectMake(0, 0, 225, 45)];
	        [self.refreshBtn setTitle:@"Refresh 0" forState:UIControlStateNormal];
	        [self.refreshBtn addTarget:delegate action:@selector(refreshPropOperationView:) forControlEvents:UIControlEventTouchUpInside];
	        [self addSubview:self.refreshBtn];
	        
	        // Prop close button. Bind to delegate's closePropOperationView:.
	        self.closeBtn = [UIButton buttonWithType:UIButtonTypeCustom];
	        [self.closeBtn setFrame:CGRectMake(0, 100, 45, 45)];
	        [self.closeBtn setImage:[UIImage imageNamed:@"close.png"] forState:UIControlStateNormal];
	        [self.closeBtn addTarget:delegate action:@selector(closePropOperationView:) forControlEvents:UIControlEventTouchUpInside];
	        [self addSubview:self.closeBtn];
	        
	    }
	    return self;
	}
	 
	 
	-(void) dealloc {
	    [_backgroundImageView release];
	    [_refreshBtn release];
	    [_closeBtn release];
	    
	    [super dealloc];
	}
	 
	@end

2.3）代码：`StoreViewCtrler.h`
`StoreViewCtrler` 拥有 `CPPropOperationView` 的实例，并实现 `CPPropOperationViewDelegate`。

	#import <UIKit/UIKit.h>
	#import "CPPropOperationView.h"
	 
	@interface StoreViewCtrler : UIViewController <CPPropOperationViewDelegate> {
	}
	@property (nonatomic, retain) CPPropOperationView* propOperationView;
	@end

2.4）代码：`StoreViewCtrler.m`

在这里，实现了 `CPPropOperationViewDelegate` 的 `refreshPropOperationView:` 和 `closePropOperationView:` 这两个方法。并通过 `self.propOperationView.delegate = self;` 将代理设为自己。那么点击 `self.propOperationView` 的 `refreshBtn` 和 `closeBtn` 将分别调用这里实现的 `refreshPropOperationView:` 和 `closePropOperationView:` 这两个方法。

	#import "StoreViewCtrler.h"
	#import "CPPropOperationView.h"
	 
	@implementation StoreViewCtrler
	@synthesize magiboxOperationView;
	 
	 
	#pragma mark - CPPropOperationViewDelegate 
	-(void) refreshPropOperationView:(id)sender { 
	     self.backgroundImageView.image = [UIImage imageNamed:@"bg2.png"];
	     [self.propOperationView.refreshBtn setTitle:[NSString stringWithFormat:@"Refresh 1"] forState:UIControlStateNormal]; 
	}
	 
	-(void) closePropOperationView:(id)sender {
	     self.propOperationView.hidden = YES;
	}
	 
	#pragma mark - View lifecycle 
	-(void) viewWillAppear:(BOOL)animated {
	    [super viewWillAppear:animated];
	     
	    if (self.propOperationView == nil) {
	        self.propOperationView = [[[CPPropOperationView alloc] initWithFrame:CGRectMake(0, 0, 300, 300)] autorelease];
	        self.propOperationView.delegate = self;
	        [self.view addSubview:self.propOperationView];
	    }
	    self.propOperationView.hidden = NO;
	}
	 
	- (void)viewDidUnload {
	    [super viewDidUnload];
	    
	    // Release any retained subviews of the main view.
	    self.magiboxOperationView = nil;
	}
	 
	-(void) dealloc {
	    [magiboxOperationView release];
	    
	    [super dealloc];
	}
	 
	@end