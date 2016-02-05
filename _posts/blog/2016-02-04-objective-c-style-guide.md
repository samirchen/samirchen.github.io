---
layout: post
title: Objective-C 编码风格指南
description: 保证自己的代码遵循团队统一的编码规范是一个码农的基本节操，能够进入一个有统一编码规范的团队则是一个码农的福气。
category: blog
tag: iOS, Objective-C, Swift
---


## 背景

本文主要是对以下几个编码规范的整理：

- [The official raywenderlich.com Objective-C style guide][3]
- [Github Objective-C style guide][4]


这里有些关于编码风格 Apple 官方文档，如果有些东西没有提及，可以在以下文档来查找更多细节：

- [The Objective-C Programming Language][6]
- [Cocoa Fundamentals Guide][7]
- [Coding Guidelines for Cocoa][8]
- [iOS App Programming Guide][9]




## 语言

使用美式英语。别用拼音。

**推荐：**


```objc
UIColor *myColor = [UIColor whiteColor];
```

**不推荐：**


```objc
UIColor *myColour = [UIColor whiteColor];
UIColor *woDeYanSe = [UIColor whiteColor];
```


## 代码结构

使用 `#pragma mark -` 根据「代码功能类别」、「protocol/delegate 方法实现」等依据对代码进行分块组织。代码的组织顺序从整体上尽量遵循我们的认知顺序，组织规范如下：

```objc
// 先描述这个类是什么，它的属性有什么。
// 每个属性的 getter 方法在前，setter 方法在后。属性的 getter/setter 方法的顺序与属性声明顺序一致。
#pragma mark - Property

- (id)customProperty {}
- (void)setCustomProperty:(id)value {}

// 再描述这个类的生命周期，从出生到消亡。
// 按照生命周期的顺序来排序相关方法。
#pragma mark - Lifecycle

- (instancetype)init {}
- (void)viewDidLoad {}
- (void)viewWillAppear:(BOOL)animated {}
- (void)viewDidAppear:(BOOL)animated {}
- (void)viewWillDisappear:(BOOL)animated {}
- (void)viewDidDisappear:(BOOL)animated {}
- (void)didReceiveMemoryWarning {}
- (void)dealloc {}

// 接着描述这个类的响应方法，能做哪些交互。
// 比如：按钮点击的响应方法、手势的响应方法等等。
#pragma mark - Action

- (IBAction)submitData:(id)sender {}

// 然后描述这个类公共方法。
#pragma mark - Public

- (void)publicMethod {}

// 然后描述这个类私有方法。
#pragma mark - Private

- (void)privateMethod {}


// 接下来描述这个类实现的 Protocol/Delegate 的方法。
// 先放自定义的 Protocol/Delegate 方法，后放官方提供的 Protocal/Delegate 方法。
#pragma mark - Protocol Conformance
#pragma mark - UITextFieldDelegate
#pragma mark - UITableViewDataSource
#pragma mark - UITableViewDelegate

// 然后是对继承的父类中方法重载。
// 先发自定义的父类方法重载，后方官方父类的方法重载。
#pragma mark - Superclass Overridden

- (void)someOverriddenMethod {}

#pragma mark - NSObject

- (NSString *)description {}
```

代码如流水一样，去叙述一个类。


## 空格

* 使用 Tab(4 个空格) 来做代码缩进，不要用空格。
* 方法大括号和其他大括号(`if`/`else`/`switch`/`while` 等)总是在同一行打开，但在新的一行关闭。

**推荐：**


```objc
if (user.isHappy) {
  //Do something
} else {
  //Do something else
}
```

**不推荐：**


```objc
if (user.isHappy)
{
    //Do something
}
else {
    //Do something else
}
```

* 在两个方法之间应该间隔一行且只有一行，这样在视觉上更清晰。在方法内可以为分隔不同的功能代码而空行，但通常都会把具有特定功能的代码抽出来成为一个新方法。
* 优先使用 auto-synthesis。如果有必要 `@synthesize` 和 `@dynamic` 的声明应该在实现代码中各占一行。
* 在调用方法时，避免以冒号对齐的格式来排版。因为有时前缀较长或者包含 Block 时会使得这种方式排版的代码易读性很差。 

**推荐：**


```objc
// blocks are easily readable
[UIView animateWithDuration:1.0 animations:^{
  // something
} completion:^(BOOL finished) {
  // something
}];
```

**不推荐：**


```objc
// colon-aligning makes the block indentation hard to read
[UIView animateWithDuration:1.0
                 animations:^{
                     // something
                 }
                 completion:^(BOOL finished) {
                     // something
                 }];
```

## 注释


当你写代码注释时，需要注意你的注释是解释为什么要有这段代码。一段注释要确保跟代码一致更新，否则就删掉。


一般避免使用块注释，这样占用空间太大，代码应该尽量做到自解释，代码即注释。*当然，也有例外：你的注释是为了生成文档用。*


## 命名

你可能是从 Java、Python、C++ 或是其他语言转过来的，但是来到 Objective-C 这底盘，请遵守苹果的命名规范，这样你才能使得自己的代码与周边和谐统一，尤其注释 [memory management rules](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html) ([NARC](http://stackoverflow.com/a/2865194/340508)) 相关的命名规范.

长的，描述性的方法和变量命名是好的，这使得代码更容易被读懂。

**推荐：**


```objc
UIButton *settingsButton;
```

**不推荐：**


```objc
UIButton *setBut;
```


在类名和常量名上应该使用两个或三个字母的前缀（比如：CX、TB 等等）。但是在 Core Data 实体命名时应该省略前缀。

常量应该使用驼峰式命名规则，所有的单词首字母大写，并加上与类名有关的前缀。


**推荐：**


```objc
static NSTimeInterval const RWTTutorialViewControllerNavigationFadeAnimationDuration = 0.3;
```

**不推荐：**


```objc
static NSTimeInterval const fadetime = 1.7;
```

属性也是使用驼峰式命名规则，但首单词的首字母小写。对属性使用 auto-synthesis，而不是手动编写 @synthesize 语句，除非你有一个好的理由。

**推荐：**


```objc
@property (strong, nonatomic) NSString *descriptiveVariableName;
```

**不推荐：**


```objc
id varnm;
```

### 下划线

当使用属性时，用 `self.` 来访问，这就意味着所有的属性都很有辨识度，因为他们前面有 `self.`。

但是有 2 个特列：

- 在属性的 getter/setter 方法中必须使用下划线，(i.e. _variableName)。
- 在类的初始化和销毁方法中，有时为了避免属性的 getter/setter 方法的负作用，可以使用下划线。

局部变量不要包含下划线。

## 方法

在方法签名中，应该在方法类型(-/+ 符号)之后有一个空格。在方法各段之间应该也有一个空格(符合 Apple 的风格)。在参数之前应该包含一个描述性的关键字来描述参数。

`and` 这个词的用法应该保留，它不应该用于多个参数之间。

**推荐：**

```objc
- (void)setExampleText:(NSString *)text image:(UIImage *)image;
- (void)sendAction:(SEL)aSelector to:(id)anObject forAllCells:(BOOL)flag;
- (id)viewWithTag:(NSInteger)tag;
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height;
```

**不推荐：**


```objc
-(void)setT:(NSString *)text i:(UIImage *)image;
- (void)sendAction:(SEL)aSelector :(id)anObject :(BOOL)flag;
- (id)taggedView:(NSInteger)tag;
- (instancetype)initWithWidth:(CGFloat)width andHeight:(CGFloat)height;
- (instancetype)initWith:(int)width and:(int)height;  // Never do this.
```

## 变量

变量尽量以描述性的方式来命名。除了在 `for()` 循环中，应该尽量避免单个字符的变量命名。

表示指针的星号应该和变量名在一起，比如：应该是 `NSString *text`，而不是 `NSString* text` 或者 `NSString * text`，除了一些特别的情况。

应该使用私有属性，而不要再使用实例变量了。这样可以保持代码的一致性。

除了在一些初始化方法(`init`, `initWithCoder:`, etc…)、销毁方法(`dealloc`)和自定义的 setters/getters 方法中外，不要直接使用下划线的方式访问实例变量。详情参见[这里](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html#//apple_ref/doc/uid/TP40004447-SW6)。

**推荐：**


```objc
@interface RWTTutorial : NSObject

@property (strong, nonatomic) NSString *tutorialName;

@end
```

**不推荐：**


```objc
@interface RWTTutorial : NSObject {
  NSString *tutorialName;
}
```


## 属性特性

所有的属性特性应该显式地列出来，有助于新手阅读代码。属性特性的顺序应该是：storage、atomicity。与在 Interface Builder 连接 UI 元素时自动生成代码一致。

**推荐：**


```objc
@property (weak, nonatomic) IBOutlet UIView *containerView;
@property (strong, nonatomic) NSString *tutorialName;
```

**不推荐：**


```objc
@property (nonatomic, weak) IBOutlet UIView *containerView;
@property (nonatomic) NSString *tutorialName;
```

具有值拷贝类型特定的属性(如：NSString)应该优先使用 `copy` 而不是 `strong`。这是因为即使你声明一个 `NSString` 类型的属性，有人也可能传入一个 `NSMutableString` 的实例，然后在你没有注意的情况下修改它。

**推荐：**


```objc
@property (copy, nonatomic) NSString *tutorialName;
```

**不推荐：**


```objc
@property (strong, nonatomic) NSString *tutorialName;
```

## 点符号语法

点符号语法是对方法调用语法很方便的一种封装。在返回属性时，使用点符号语法，属性的 getter/setter 方法也能确保被调用。更多信息阅读[这里](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/ProgrammingWithObjectiveC/EncapsulatingData/EncapsulatingData.html)。

我们应该总是使用点符号预发类访问或者修改属性，因为它使得代码更加简洁。`[]` 则应该用在其他场景下。

**推荐：**

```objc
NSInteger arrayCount = [self.array count];
view.backgroundColor = [UIColor orangeColor];
[UIApplication sharedApplication].delegate;
```

**不推荐：**

```objc
NSInteger arrayCount = self.array.count;
[view setBackgroundColor:[UIColor orangeColor]];
UIApplication.sharedApplication.delegate;
```

## 字面值

在创建 `NSString`、`NSDictionary`、`NSArray` 和 `NSNumber` 对象时，应该使用字面值语法。尤其需要注意创建 `NSArray` 和 `NSDictionary` 对象时，不能传入 `nil`，否则会造成 crash。

**推荐：**


```objc
NSArray *names = @[@"Brian", @"Matt", @"Chris", @"Alex", @"Steve", @"Paul"];
NSDictionary *productManagers = @{@"iPhone": @"Kate", @"iPad": @"Kamal", @"Mobile Web": @"Bill"};
NSNumber *shouldUseLiterals = @YES;
NSNumber *buildingStreetNumber = @10018;
```

**不推荐：**


```objc
NSArray *names = [NSArray arrayWithObjects:@"Brian", @"Matt", @"Chris", @"Alex", @"Steve", @"Paul", nil];
NSDictionary *productManagers = [NSDictionary dictionaryWithObjectsAndKeys: @"Kate", @"iPhone", @"Kamal", @"iPad", @"Bill", @"Mobile Web", nil];
NSNumber *shouldUseLiterals = [NSNumber numberWithBool:YES];
NSNumber *buildingStreetNumber = [NSNumber numberWithInteger:10018];
```

## 常量

比起硬编码字符串或数字的形式，我们应该常量来定义复用型变量，因为常量更容易被修改，而不需要我们 find + replace。使用常量时，我们应该使用 `static` 而不是 `#define` 一个类型不明的宏。

**推荐：**


```objc
static NSString * const RWTAboutViewControllerCompanyName = @"RayWenderlich.com";

static CGFloat const RWTImageThumbnailHeight = 50.0;
```

**不推荐：**


```objc
#define CompanyName @"RayWenderlich.com"

#define thumbnailHeight 2
```

## 枚举类型

当使用枚举时，我们要用 `NS_ENUM()` 而不是 `enum`。

**例如：**

```objc
typedef NS_ENUM(NSInteger, RWTLeftMenuTopItemType) {
  RWTLeftMenuTopItemMain,
  RWTLeftMenuTopItemShows,
  RWTLeftMenuTopItemSchedule
};
```

你可以显示的赋值：

```objc
typedef NS_ENUM(NSInteger, RWTGlobalConstants) {
  RWTPinSizeMin = 1,
  RWTPinSizeMax = 5,
  RWTPinCountMin = 100,
  RWTPinCountMax = 500,
};
```


**不推荐：**


```objc
enum GlobalConstants {
  kMaxPinSize = 5,
  kMaxPinCount = 500,
};
```


## Case 语句

除非编译器强制要求，一般在 Case 语句中是不需要加括号的。当一个 Case 语句包含多行代码，应该加上括号。

```objc
switch (condition) {
  case 1:
    // ...
    break;
  case 2: {
    // ...
    // Multi-line example using braces
    break;
  }
  case 3:
    // ...
    break;
  default: 
    // ...
    break;
}

```

<!-- There are times when the same code can be used for multiple cases, and a fall-through should be used.  A fall-through is the removal of the 'break' statement for a case thus allowing the flow of execution to pass to the next case value.  A fall-through should be commented for coding clarity. -->

如果一段代码被多个 Case 语句共享执行，那就要用 fall-through，即在 Case 语句中删除 `break` 语句，让代码能够执行到下一个 Case 中去，为了代码清晰明了，用了 fall-through 时需要注释一下。

```objc
switch (condition) {
  case 1:
    // ** fall-through! **
  case 2:
    // code executed for values 1 and 2
    break;
  default: 
    // ...
    break;
}

```

在 Swith 中使用枚举类型时，是不需要 `default` 语句的，例如：

```objc
RWTLeftMenuTopItemType menuType = RWTLeftMenuTopItemMain;

switch (menuType) {
  case RWTLeftMenuTopItemMain:
    // ...
    break;
  case RWTLeftMenuTopItemShows:
    // ...
    break;
  case RWTLeftMenuTopItemSchedule:
    // ...
    break;
}
```


## 私有属性

<!-- Private properties should be declared in class extensions (anonymous categories) in the implementation file of a class. Named categories (such as `RWTPrivate` or `private`) should never be used unless extending another class.   The Anonymous category can be shared/exposed for testing using the <headerfile>+Private.h file naming convention. -->

私有属性应该在类的实现文件(xxx.m)中的匿名扩展(Anonymous Category)中声明。除非是要去扩展一个类，否则不要使用命名扩展(Named Category)。如果你要测试私有属性，你可以通过 `<headerfile>+Private.h` 的方式把私有属性暴露给测试人员。

**For Example:**

```objc
@interface RWTDetailViewController ()

@property (strong, nonatomic) GADBannerView *googleAdView;
@property (strong, nonatomic) ADBannerView *iAdView;
@property (strong, nonatomic) UIWebView *adXWebView;

@end
```

## 布尔值

Objective-C 使用 `YES` 和 `NO` 作为 BOOL 值。因此，`true` 和 `false` 只应该在 CoreFoundation、C、C++ 代码中使用。由于 `nil` 会被解析为 `NO`，所以没有必要在条件语句中去比较它。 另外，永远不要拿一个对象和 `YES` 比较，因为 `YES` 被定义为 1 并且 `BOOL` 值最多 8 bit。

这时为了在不同代码中保持一致性和简洁性。

**推荐：**


```objc
if (someObject) {}
if (![anotherObject boolValue]) {}
```

**不推荐：**


```objc
if (someObject == nil) {}
if ([anotherObject boolValue] == NO) {}
if (isAwesome == YES) {} // Never do this.
if (isAwesome == true) {} // Never do this.
```

如果一个 `BOOL` 类型的属性是形容词，那么它的命名可以省略掉 “is” 前缀，但是我们还是需要给它指定惯用的 getter 方法名，例如:

```objc
@property (assign, getter=isEditable) BOOL editable;
```

更多内容详见：[Cocoa Naming Guidelines](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CodingGuidelines/Articles/NamingIvarsAndTypes.html#//apple_ref/doc/uid/20001284-BAJGIIJE)。



## 条件语句

<!-- Conditional bodies should always use braces even when a conditional body could be written without braces (e.g., it is one line only) to prevent errors. These errors include adding a second line and expecting it to be part of the if-statement. Another, [even more dangerous defect](http://programmers.stackexchange.com/a/16530) may happen where the line "inside" the if-statement is commented out, and the next line unwittingly becomes part of the if-statement. In addition, this style is more consistent with all other conditionals, and therefore more easily scannable. -->

条件语句应该使用大括号包围，即使能够不用时（比如条件代码只有一行）也不要省略大括号，这样可以最大可能的避免出错（比如条件语句不小心被注释了），同时也保持了大括号的使用风格一致。


**推荐：**

```objc
if (!error) {
  return success;
}
```

**不推荐：**

```objc
if (!error)
  return success;
```

或者

```objc
if (!error) return success;
```

### 三元操作符

只有在能提高代码清晰性和可读性的情况下，才应该使用三元操作符 `?:`。单个条件判断时可以用到它，多个条件判断时还是用 `if` 来提高代码可读性吧。一般来说，使用三元操作符最好的场景是根据条件来赋值的时候。

非布尔类型的变量与某对象比较时最好加上括号来提高代码可读性，如果被比较的变量是布尔类型那就不用括号了。

**推荐：**

```objc
NSInteger value = 5;
result = (value != 0) ? x : y;

BOOL isHorizontal = YES;
result = isHorizontal ? x : y;
```

**不推荐：**

```objc
result = a > b ? x = c > d ? c : d : y;
```

## 初始化方法

Init 方法应该遵循 Apple 生成代码模板的命名规则。返回类型应该使用 `instancetype` 而不是 `id`。

```objc
- (instancetype)init {
  self = [super init];
  if (self) {
    // ...
  }
  return self;
}
```


## 类构造方法

当使用类构造方法时，应该返回的类型是 `instancetype` 而不是 `id`。这样确保编译器正确地推断结果类型。 

```objc
@interface Airplane
+ (instancetype)airplaneWithType:(RWTAirplaneType)type;
@end
```

查看更多关于 instancetype 的信息：[NSHipster.com](http://nshipster.com/instancetype/)。

## CGRect 方法

当访问 CGRect 的 `x`、`y`、`width`、`height` 属性时，总是使用 [`CGGeometry` functions](http://developer.apple.com/library/ios/#documentation/graphicsimaging/reference/CGGeometry/Reference/reference.html) 相关的函数，而不是直接从结构体访问。

> All functions described in this reference that take CGRect data structures as inputs implicitly standardize those rectangles before calculating their results. For this reason, your applications should avoid directly reading and writing the data stored in the CGRect data structure. Instead, use the functions described here to manipulate rectangles and to retrieve their characteristics.

**推荐：**


```objc
CGRect frame = self.view.frame;

CGFloat x = CGRectGetMinX(frame);
CGFloat y = CGRectGetMinY(frame);
CGFloat width = CGRectGetWidth(frame);
CGFloat height = CGRectGetHeight(frame);
CGRect frame = CGRectMake(0.0, 0.0, width, height);
```

**不推荐：**


```objc
CGRect frame = self.view.frame;

CGFloat x = frame.origin.x;
CGFloat y = frame.origin.y;
CGFloat width = frame.size.width;
CGFloat height = frame.size.height;
CGRect frame = (CGRect){ .origin = CGPointZero, .size = frame.size };
```

## 黄金路径

当使用条件语句编写逻辑时，左手的代码应该是 "golden" 或 "happy" 路径。也就是说，不要嵌套多个 `if` 语句，即使写多个 `return` 语句也是 OK 的。

**推荐：**


```objc
- (void)someMethod {
  if (![someOther boolValue]) {
	return;
  }

  //Do something important
}
```

**不推荐：**


```objc
- (void)someMethod {
  if ([someOther boolValue]) {
    //Do something important
  }
}
```

## 错误处理

当方法通过引用来返回一个错误参数时，应该判断返回值而不是错误变量。

**推荐：**

```objc
NSError *error;
if (![self trySomethingWithError:&error]) {
  // Handle Error
}
```

**不推荐：**

```objc
NSError *error;
[self trySomethingWithError:&error];
if (error) {
  // Handle Error
}
```

<!-- Some of Apple’s APIs write garbage values to the error parameter (if non-NULL) in successful cases, so switching on the error can cause false negatives (and subsequently crash). -->

因为有些 Apple 的 API 在方法调用成功的情况下也会往 error 中写入垃圾值，这时候如果根据 error 来做判断就会得到不正确的结果，甚至 crash。


## 单例

单例对象应该使用线程安全的方式来创建共享实例。


```objc
+ (instancetype)sharedInstance {
  static id sharedInstance = nil;

  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    sharedInstance = [[self alloc] init];
  });

  return sharedInstance;
}
```

这样会防止 [possible and sometimes prolific crashes](http://cocoasamurai.blogspot.com/2011/04/singletons-your-doing-them-wrong.html)。


## 换行符

换行符主要是在提高打印和网上阅读时的代码可读性时显得很重要。

例如：

```objc
self.productsRequest = [[SKProductsRequest alloc] initWithProductIdentifiers:productIdentifiers];
```

一行较长的代码最好能换行再加一个 Tab。

```objc
self.productsRequest = [[SKProductsRequest alloc] 
 	initWithProductIdentifiers:productIdentifiers];
```
 


## Xcode 工程

<!-- The physical files should be kept in sync with the Xcode project files in order to avoid file sprawl. Any Xcode groups created should be reflected by folders in the filesystem. Code should be grouped not only by type, but also by feature for greater clarity. -->

<!-- When possible, always turn on "Treat Warnings as Errors" in the target's Build Settings and enable as many [additional warnings](http://boredzo.org/blog/archives/2009-11-07/warnings) as possible. If you need to ignore a specific warning, use [Clang's pragma feature](http://clang.llvm.org/docs/UsersManual.html#controlling-diagnostics-via-pragmas). -->

物理文件应该与 Xcode 项目目录保持同步来避免文件管理杂乱。创建任何 Xcode group 应该与文件系统中的文件夹保持映射。代码分类除了以类型分类，从大的方面上也应该以功能分类。

如果可以的话，打开 Xcode 的 `Treat Warnings as Errors` 来降低对 warning 的容忍度。如果有时候确实要忽略某一个 warning，你可以使用 [Clang's pragma feature](http://clang.llvm.org/docs/UsersManual.html#controlling-diagnostics-via-pragmas)。









[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/objective-c-style-guide
[3]: https://github.com/raywenderlich/objective-c-style-guide#language
[4]: https://github.com/github/objective-c-style-guide
[5]: http://www.jianshu.com/p/8b76814b3663
[6]: http://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/ObjectiveC/Introduction/introObjectiveC.html
[7]: https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CocoaFundamentals/Introduction/Introduction.html
[8]: https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CodingGuidelines/CodingGuidelines.html
[9]: http://developer.apple.com/library/ios/#documentation/iphone/conceptual/iphoneosprogrammingguide/Introduction/Introduction.html
