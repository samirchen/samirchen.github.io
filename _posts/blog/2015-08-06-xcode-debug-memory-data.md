---
layout: post
title: 在 Xcode 调试时查看内存中的数据
description: 有时候我们在调试时希望可以追踪超出作用域变量的内存地址的数据，这篇文章就简要介绍一下相关的命令。
category: blog
tag: iOS, debug, Xcode
---

## 前言

LLDB 是一个有着 REPL 的特性和 C++ ,Python 插件的开源调试器。LLDB 绑定在 Xcode 内部，存在于主窗口底部的控制台中。调试器允许你在程序运行的特定时暂停它，你可以查看变量的值，执行自定的指令，并且按照你所认为合适的步骤来操作程序的进展。想知道 LLDB 更多的命令，可以在 LLDB 窗口使用 help 命令查看。更多关于 Xcode 调试的知识可以看这篇文章：[与调试器共舞 - LLDB 的华尔兹](http://objccn.io/issue-19-2/)

在这里只列出在 Xcode 调试时查看内存中的数据的命令。

## 命令格式


	x/nfu <addr>

## 参数解释


- n，表示要显示的内存单元的个数
- f，表示显示方式, 可取如下值：
	- x 按十六进制格式显示变量
	- d 按十进制格式显示变量
	- u 按十进制格式显示无符号整型
	- o 按八进制格式显示变量
	- t 按二进制格式显示变量
	- a 按十六进制格式显示变量
	- i 指令地址格式
	- c 按字符格式显示变量
	- f 按浮点数格式显示变量
- u，表示一个地址单元的长度：
	- b 表示单字节
	- h 表示双字节
	- w 表示四字节
	- g 表示八字节
- <addr\>，表示内存地址，可以是变量名，也可以是内存地址。

## 例子



	x/16xb self
	会显示 self 指针地址内容，16 个字节，16 进制。
	
	x/8cb 0x7fc359a03040
	会显示地址 0x7fc359a03040 地址的内容，8 个字节，按字符格式显示。



[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://samirchen.com/xcode-debug-memory-data

