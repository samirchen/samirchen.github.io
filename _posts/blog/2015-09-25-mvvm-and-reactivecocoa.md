---
layout: post
title: MVVM 模式及 ReactiveCocoa
description: 说说 MVVM 模式及 ReactiveCocoa。
category: blog
tag: iOS, Objective-C, MVVM, ReactiveCocoa
---


## 为什么要 MVVM
MVVM 的最大作用是为了解决 Massive View Controller 的问题，为 Cocoa MVC 模式下臃肿的 ViewController 瘦身。

MVVM 在 View(ViewController) 与 Model 直接加了一层 ViewModel，也因此带来了问题：当 Model 的数据发生变化不能直接通知 View(ViewController)？



Model(Entity + Service)

ViewModel(Between Model and View(ViewController))

View(ViewController)


还可以去 Model 化。直接 ViewModel 吃掉了 Model。



还是需要 Model 和 ViewModel 区分的。ViewModel 包好了页面需要展示的信息，以及对应的 Models，在这些 Models 里包含了页面展示的信息和不展示的信息，一些不展示的信息对于逻辑处理是不可缺少的，常见的比如 userID。


[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://samirchen.com/mvvm-and-reactivecocoa/
