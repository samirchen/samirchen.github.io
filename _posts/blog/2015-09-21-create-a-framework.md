---
layout: post
title: 创建一个 Framework 项目 
description: 简单介绍一下创建一个 Framework 项目的流程。
category: blog
tag: iOS
---

##什么是 Framework

Framework 是 Mac OS/iOS 平台用来打包代码的一种方式，它可以将代码文件、头文件和有关的资源文件一起打包提供给开发者使用。

谈到 Framework，不妨了解了解库的概念，所谓的库（Library）即使一段编译好的二进制代码，再配上描述库接口的头文件从而可以提供给其他开发者使用。在使用库的时候需要对库的代码进行 Link，而 Link 有两种方式：静态和动态，从而产生了静态链接库和动态链接库的概念。

静态库的常见形式包括：Windows 下的 `.lib` 文件，Linux 和 Mac 下的 `.a` 文件。它的 Link 方式是在编译的时候直接拷贝一份库代码，复制到目标程序里，这段代码在目标程序中就不会再变动了。这样做的**好处**是编译完后，库文件实际就没啥用了，目标程序没有外部依赖，直接运行即可；**缺点**是会造成目标程序的体积增大。

动态库的常见形式包括：Windows 下的 `.dll` 文件，Linux 下的 `.so` 文件，Mac 下的 `.dylib` 文件。与静态库不同，动态库在编译时并不会被拷贝到目标程序中，目标程序中只会存储指向动态库的引用。等到程序运行时，动态库才会被真正加载进来。这样做的**好处**是不需要拷贝库代码到目标程序中，不会增大目标程序的体积，而且同一份库可以被多个程序使用（因此动态库也被叫做共享库）。此外，运行时才载入的特征也使得我们可以随时对动态库进行替换，而不需要重新编译代码，这点对开发者来说具有很大的吸引力。动态库的**缺点**主要是在载入时会需要一些性能消耗，而且动态库也使得目标程序需要依赖于外部环境，当目标程序依赖的动态库在外部环境中缺失或版本不符时，就会导致程序执行出错。

回过头来再说 Framework，它其实就是一种动态链接库，只不过它除了打包代码外还能打包资源文件。在 iOS 8 发布时，苹果开放了对动态 Framework 的支持，这应该是苹果为支持 Extension 这一特性而做出的选择（Extension 和 App 是两个分开的可执行文件，它们之间共享代码，所以需要 Framework 支持）。但是这种动态 Framework 和系统自带的 Framework 还是有区别的，系统的 Framework 是不需要我们拷贝到 App 里的，但是我们自己的 Framework 还是需要拷贝到 App 里的。虽然可以使用动态 Framework 了，但是从服务器动态更新动态库的做法在 iOS 上是不被苹果支持的，所以是无法实现的，因为 Sandbox 会验证动态库的签名，是从服务器加载的动态库是无法签名的。

如我们所知，跟着 iOS 8 和 Xcode 6 一起发布的还有 Swift，现在 Swift 是不支持静态库的，只能支持动态 Framework。造成这个问题的原因主要是 Swift 的 Runtime 没有被包含在 iOS 系统中，而是会打包进 App 中（这也是造成 Swift App 体积大的原因），静态库会导致最终的目标程序中包含重复的 Runtime。同时拷贝 Runtime 这种做法也会导致在纯 ObjC 的项目中使用 Swift 库出现问题。苹果声称等到 Swift 的 Runtime 稳定之后会被加入到系统当中，到时候这个限制就会被去除了。

现在通过 Xcode 6 及以上的版本我们已经可以自己创建 Cocoa Touch Framework 了。现在就来讲一讲。


##创建一个 Framework 项目

1) 在 Xcode 中，File -> New -> Project -> Framework & Library -> Cocoa Touch Framework 来创建项目。

![image](../../images/create-a-framework/cocoa_touch_framework.png)

2) 向项目中添加代码。（CXTextField.h, CXTextField.m）

![image](../../images/create-a-framework/framework-project-tree.png)

3) 在 CXUIKit.h 文件中添加公共代码的头文件：`#import <CXUIKit/CXTextField.h>`。

代码如下：

	#import <UIKit/UIKit.h>	
	//! Project version number for CXUIKit.
	FOUNDATION_EXPORT double CXUIKitVersionNumber;
	//! Project version string for CXUIKit.
	FOUNDATION_EXPORT const unsigned char CXUIKitVersionString[];
	// In this header, you should import all the public headers of your framework using statements like #import <CXUIKit/PublicHeader.h>
	#import <CXUIKit/CXTextField.h>

4) 开放公用代码的头文件。把需要曝光给外部的头文件放到 Targets -> Build Phases -> Headers -> Public 栏中。

![image](../../images/create-a-framework/add-headers.png)

接下来，应该就能编译成功了。但是需要注意的是，你选择的编译的目标平台不一样编译得到的 CXUIKit.framework 是不一样的。

![image](../../images/create-a-framework/compile-target.png)

##创建一个测试 Framework 的 Demo 项目

1) 在 Xcode 中，File -> New -> Project -> Application -> Single View Application 创建一个测试 Framework 的 Demo App：CXUIKitDemo。将上一节中创建的 Framework 项目的 CXUIKit.xcodeproj 文件拖到 CXUIKitDemo 中，位置如图所示。

![image](../../images/create-a-framework/demo-project.png)

2) 在 CXUIKitDemo 项目的 Targets -> Build Phases -> Target Dependencies 中添加 CXUIKit Framework。

![image](../../images/create-a-framework/demo-project-build-phases.png)

##在 App 项目中使用 Framework















[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://samirchen.com/create-a-framework/
[3]: http://www.knowstack.com/framework-vs-library-cocoa-ios/