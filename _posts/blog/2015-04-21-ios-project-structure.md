---
layout: post
title: iOS项目的目录结构
description: 介绍iOS项目开发中的目录结构设计以及背后的一些想法。
category: blog
tag: iOS, project structure
---



##结论
不废话，我现在通常采用的目录结构如下：

```
|—MyProject
    |—ignore-folder // 放置不想同步到代码服务器上的内容，通常包括一些体积太大、经常变动、对项目运行影响不大的文件。需要在该目录下添加 .gitignore 对本目录做一些设置。
        |—readme.log // 因为 ignore-folder 目录下的内容都是不会同步到代码服务器上的，所以最好加一个 log 文件记录一下你在该目录的操作。
        |—3rdparty // 比如，一些不能用 CocoaPods 管理也不想同步到代码服务器上的第三方库。
        |—data // 比如，一些经常会变动的、自己的测试数据文件。
    |—Utility // 自己实现的一些通用性较好的功能代码，这些代码有比较好的接口且与本项目不存在耦合，可直接复用于其他项目。
    |—Common // 本项目的一些全局性代码，这些代码通常与本项目的业务逻辑存在一些耦合，所以不放在 Utility 目录中。
    |—Feature // 本项目的功能模块目录，该目录下将项目的功能划分为多个模块，每个模块穿透 MVC，可以独立划分出去。当然，在模块下你不采用 MVC，采用 MVVM 或其他架构方式也没问题的。
        |—Base // 定义本项目中各种 Controller、View、Model 的基础类或基础接口。
            |—Controller
            |—View
            |—Model
        |—Main
            |—Controller
            |—View
            |—Model
        |—User
            |—Controller
            |—View
            |—Model
    |—Resource // 本项目的资源目录，放置图片、音频等资料。
        |—Image
        |—Sound
|—Pods // 采用 CocoaPods 管理的第三方库。
```




##目录结构的进化

###形式一

```
|—MyProject
    |—ignore-folder
        |—readme.log
        |—3rdparty
        |—data 
    |—Utility
    |—Common
    |—Service
        |—LocalService // 封装在 DAO 层之上，直接对接业务逻辑层，提供本地数据服务。
            |—DAO // 封装本地数据库访问层的相关代码。
        |—WebService // 对接业务逻辑层，提供网络数据服务。
    |—Model // 封装项目中的实体类。
    |—View
    |—Controller
    |—Resource
        |—Image
        |—Sound
|—Pods
```

上面的目录结构主要特点是基于 MVC 的架构构建的。Model 层主要封装一些数据对象实体，它们的实例将在项目的数据流中流动；View 层主要放一些控件，被 Controller 层用来展示；Controller 层就是一些 ViewController，获得数据 Model，用 View 展示出来；Service 层则为 Controller 层提供本地或网络的数据服务。

以前跟同事一起开发的时候会分层来分工，比如，当对项目抽象完毕、设计好数据库后，就会由同学 A 负责映射数据库的 Model 层的开发，然后提供相应的 DAO 层、LocalService 层的接口，一般就是常见的增删查改操作接口，当负责 Controller 层的同学 B 需要本地数据时，则去调用同学 A 提供的 LocalService 层的接口。这时问题就出现了，由于同学 A 开发 LocalService 层的接口时，并没有接触到上层的业务逻辑，他设计接口时只能充分发挥他自己的想象力，一般也就是提供常见的增删查改操作，这些接口与业务逻辑对接时，常常会发现要么冗余，提供的接口根本就用不上，要么对于复杂点的业务逻辑支持不够，想复杂一点操作数据，现有的接口却支持不了。所以，就需要考虑换一种分工方式，采用基于 Feature 的方式进行分工了，基于这点，项目的目录结构也逐渐演变成下面要讲的形式了。


###形式二
下面的目录结构就是文章开头所说的我现在通常采用的目录结构。这种形式的主要特点就是更好地支持基于 Feature 进行模块划分和任务指派。在每个 Feature 下，对应的开发人员需要穿透 MVC 整个层次来完成这个功能模块的开发，那么他对于各层的接口也能更高效地开发，不需要的接口不用写，复杂的接口能写的更高效。甚至，开发人员可以根据 Feature 的特点采用更适合的架构，比如 MVVM 架构等等，这样更具灵活性。但是需要关注的是，还是需要提供一定规范限制，保持每个 Feature 下代码结构的清晰，这样也利于同事的查阅、修改和调用。

```
|—MyProject
    |—ignore-folder
        |—readme.log
        |—3rdparty
        |—data
    |—Utility
    |—Common
    |—Feature
        |—Base
            |—Controller
            |—View
            |—Model
        |—Main
            |—Controller
            |—View
            |—Model
        |—User
            |—Controller
            |—View
            |—Model
    |—Resource
        |—Image
        |—Sound
|—Pods
```


###基本原则
上面的 iOS 项目目录结构不一定适合所有人的想法，关键看你希望用一个结构解决什么问题。就我自己而言，我是希望通过一个良好的项目结构去达到两个目的：

- 1）使项目更适合于团队开发，能够降低耦合、便于任务的划分和代码的整合管理。
- 2）使项目能够积累出更多可复用的代码和架构。

这个结构会在不断遇到问题解决问题的过程中权衡、进化，在这个过程最重要的是能够保持：

- 1）主干简洁。主干上防止过度划分，过度划分会让代码放在这个目录下也可以，放在另一个目录下好像也行，容易混乱。
- 2）分支开放。不对过于细节的分支做严格规范，可以发挥大家的灵活性和创造性。




##关于Xcode的文件夹

###Group 和 Folder Reference 的区别
说完目录结构，插一点小话题，说说 Xcode 的文件夹。Xcode 项目的文件夹有 Group 和 Folder Reference 之分。它们的区别在 [Xcode Groups vs. Folder References][3] 这篇文章里有详细的讲述。

Group 的缺点如下：

- 1）Xcode 会为每个文件创建一个 reference，存储在 project.pbxproj 文件中，当有多个 target 时，每个文件的 reference 会被复制多个，这就会大大增加 project.pbxproj 文件的尺寸和复杂度，这对于代码版本管理来是个头疼的问题，尤其是遇到 merge conflict 的时候。
- 2）项目中 Group 的结构跟磁盘上的文件价的结构可以说是没有啥关系的，在磁盘上的一个文件在一个某个文件夹中，但是在 Xcode 的项目结构中，它可能在任何 Group 中，这样想要去找对应的文件就常常让人很晕。
- 3）如果你在 Xcode 之外直接去 Finder 里移动项目文件到不同的目录时，那么在 Xcode 中对这个文件的 reference 就会坏掉。

Group 的优点如下：

- 1）你可以选择磁盘上的文件添加到项目中，不想要的不添加就行了。
- 2）对不同的 target 对应的文件能够更好地管理，比如，你可以选择对一个 target 排除某一个文件。
- 3）当 build 的时候，Xcode 会把所有的 Group 下的文件都放到 bundle 的顶级目录，所以你调用文件时不需要制定它的具体位置，比如，你不必这样 `[UIImage imageNamed:@"Images/Icons/Go"];` ，这用这样就可以 `[UIImage imageNamed:@"Go"];`，但是这就意味着在整个项目中，你不能有同名的文件了。

Folder Reference 的有这些优点：

- 1）Xcode 只存储 folder 的 reference，这个 folder 下所有的文件和 subfolder 都会自动添加到项目中去。这会使项目文件更小更简单，代码版本管理时，产生 merge conflict 的可能就更少。
- 2）如果你在文件系统中直接对 folder 下的文件进行修改、移动或者删除，Xcode 会自动更新对应的 folder reference 来反应这些改变，这样管理项目文件也更简单。
- 3）项目的结构和 folder 在磁盘的结构是一致的，这样就不会晕菜了。
- 4）由于存在不同的 folder 路径，你就不需要担心文件重名问题了，因为在 build 的时候，文件夹结构也会被放到 bundle 中去。 

Folder Reference 的有这些缺点：

- 1）对不同的 target 的管理是个灾难，因为一个 folder 下的代码或文件，要么全一样要么全不要。当然，如果你能为不同的 target 去建立不同的 Folder Reference，这看起来也没什么不好的。
- 2）对 folder 下的文件无法隐藏，磁盘上如果这个 folder 下有这个文件，那么在项目结构中就会看到它。
- 3）在加载文件资源的时候，你必须制定全路径。也就是说，你得这样：`[UIImage imageNamed:@"Images/Icons/Go"];`。
- 4）存储在 Folder Reference 中的图片在 Interface Builder 中使用时会遇到各种问题。

在实际使用中，使用 Group 要多得多。

###Xcode 项目结构和磁盘文件结构的对应
上面说了 Group 和 Folder Reference 各自的优缺点，在项目中，我习惯上也是不使用 Folder Reference，只用 Group。在 Xcode 项目中创建 Group 的方式有两种：

- 1）创建时需要先在磁盘上创建对应的文件夹，再把文件夹拖进 Xcode 项目中对应的位置，并选择 Create groups for any added folders。这样创建的 Group 会对应着磁盘上的 Folder。
- 2）直接在 Xcode 项目中创建 Group。

对照 Xcode 项目的 MyProject.xcodeproj/project.pbxproj 文件可以看到对应着 Folder 的 Group 和直接创建的 Group 的区别就在于前者是用 path 属性去记录，后者是用 name 属性去记录。如下，WebService 是一个不对应 Folder 的 Group，Resource 是一个对应 Folder 的 Group。通过各个结点的父子关系以及 path 属性，Xcode 就能管理好每个文件的 Reference。

```
45E59EDF18BBA92C00251797 /* WebService */ = {
     isa = PBXGroup;
     children = (
          452183D3195AA18F00679F14 /* CXTaskControlService.h */,
          452183D4195AA18F00679F14 /* CXTaskControlService.m */,
     );
     name = WebService;
     sourceTree = "<group>";
};
45E59EE318BC2B4100251797 /* Resource */ = {
     isa = PBXGroup;
     children = (
          45E59EE418BC2B4100251797 /* Image */,
          45C97CA91900260A0020C517 /* Sound */,
     );
     path = Resource;
     sourceTree = "<group>";
};
```





对于第一种方式：

- `好处`：这样磁盘上的结构和 Xcode 中的项目结构就是一一对应的。让 Xcode 的目录结构能够跟磁盘文件的结构保持一致，这样让在找代码文件的时候能够更清晰。
- `坏处`：创建和管理代码文件时就要麻烦一些。当你在 Xcode 项目中把一个文件从一个目录拖动到另外一个目录中时，它在 Xcode 中显示的目录路径改变了，但是它在磁盘上的物理位置并没有发生改变，这就会造成混乱。所以，为了保持对应关系，当你要改变文件的目录时，你需要手动到 Finder 文件夹中去移动文件，再在 Xcode 项目中去删除相应的文件引用再重新添加到新的目录下。这又带来了另外一个问题，就是 git 对文件的版本管理信息会被破坏，你可能无法看到这个文件之前的版本了，这个代价可就大了。所以，如果要采用版本管理，就要好好考虑这个因素了。


对于第二种方式：

- `好处`：直接在 Xcode 中管理项目，添加、删除、移动都很方便。采用版本管理时，代码文件的 版本信息不会因为移动而丢失。
- `坏处`：Xcode 中的项目结构和磁盘上的结构不能一一对应。

根据上面的对比，

- `推荐`：对于固定的可复用代码可以采用第一种方式，因为也不会经常移动。复用时，挪动起来也好操作；对于大的目录结构，比如一级目录：Utility、Common、Feature、Resource 等目录可以采用第一种方式。其他的情况均采用第二种方式就行了。



[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://samirchen.com/ios-project-structure/
[3]: http://vocaro.com/trevor/blog/2012/10/21/xcode-groups-vs-folder-references/

