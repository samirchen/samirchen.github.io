---
layout: post
title: Xcode中使用git
description: 主要针iOS团队开发使用Xcode和git时的一些场景做一些小结。
category: blog
---

在之前写的一篇博客[git命令详解][2]里面尽量详细系统地介绍了一些git命令。这里主要针[iOS][6]团队开发使用[Xcode][5]和git时的一些场景做一些小结。

## 关于.gitignore

首先说说`.gitignore`。在 [Mac OS X][4] 上用 [Xcode][5] 做iOS开发，有一些文件通常是不应该提交到版本管理库里面去的，那就需要在项目的根目录下添加 `.gitignore` 文件，并编辑其内容，将这些文件按一定的规则添加进来。

下面的 `.gitignore` 文件是参考[Stack Overflow上关于 gitignore for Xcode Project 的讨论](http://stackoverflow.com/questions/49478/git-ignore-file-for-xcode-projects)而来，基本上总结了目前我们需要在 `.gitignore` 中设置的内容，并做了详细的说明：


	#########################
	# .gitignore file for Xcode4 / OS X Source projects
	#
	# Version 2.1
	# For latest version, see: http://stackoverflow.com/questions/49478/git-ignore-file-for-xcode-projects
	#
	# 2013 updates:
	# - fixed the broken "save personal Schemes"
	# - added line-by-line explanations for EVERYTHING (some were missing)
	#
	# NB: if you are storing "built" products, this WILL NOT WORK,
	# and you should use a different .gitignore (or none at all)
	# This file is for SOURCE projects, where there are many extra
	# files that we want to exclude
	#
	#########################
	
	#####
	# OS X temporary files that should never be committed
	#
	# c.f. http://www.westwind.com/reference/os-x/invisibles.html
	
	.DS_Store
	
	# c.f. http://www.westwind.com/reference/os-x/invisibles.html
	
	.Trashes
	
	# c.f. http://www.westwind.com/reference/os-x/invisibles.html
	
	*.swp
	
	# *.lock - this is used and abused by many editors for many different things.
	#    For the main ones I use (e.g. Eclipse), it should be excluded 
	#    from source-control, but YMMV
	
	*.lock
	
	#
	# profile - REMOVED temporarily (on double-checking, this seems incorrect; I can't find it in OS X docs?)
	#profile
	
	
	####
	# Xcode temporary files that should never be committed
	# 
	# NB: NIB/XIB files still exist even on Storyboard projects, so we want this...
	
	*~.nib
	
	
	####
	# Xcode build files -
	#
	# NB: slash on the end, so we only remove the FOLDER, not any files that were badly named "DerivedData"
	
	DerivedData/
	
	# NB: slash on the end, so we only remove the FOLDER, not any files that were badly named "build"
	
	build/
	
	
	#####
	# Xcode private settings (window sizes, bookmarks, breakpoints, custom executables, smart groups)
	#
	# This is complicated:
	#
	# SOMETIMES you need to put this file in version control.
	# Apple designed it poorly - if you use "custom executables", they are
	#  saved in this file.
	# 99% of projects do NOT use those, so they do NOT want to version control this file.
	#  ..but if you're in the 1%, comment out the line "*.pbxuser"
	
	# .pbxuser: http://lists.apple.com/archives/xcode-users/2004/Jan/msg00193.html
	
	*.pbxuser
	
	# .mode1v3: http://lists.apple.com/archives/xcode-users/2007/Oct/msg00465.html
	
	*.mode1v3
	
	# .mode2v3: http://lists.apple.com/archives/xcode-users/2007/Oct/msg00465.html
	
	*.mode2v3
	
	# .perspectivev3: http://stackoverflow.com/questions/5223297/xcode-projects-what-is-a-perspectivev3-file
	
	*.perspectivev3
	
	#    NB: also, whitelist the default ones, some projects need to use these
	!default.pbxuser
	!default.mode1v3
	!default.mode2v3
	!default.perspectivev3
	
	
	####
	# Xcode 4 - semi-personal settings
	#
	#
	# OPTION 1: ---------------------------------
	#     throw away ALL personal settings (including custom schemes!
	#     - unless they are "shared")
	#
	# NB: this is exclusive with OPTION 2 below
	xcuserdata
	
	# OPTION 2: ---------------------------------
	#     get rid of ALL personal settings, but KEEP SOME OF THEM
	#     - NB: you must manually uncomment the bits you want to keep
	#
	# NB: this *requires* git v1.8.2 or above; you may need to upgrade to latest OS X,
	#    or manually install git over the top of the OS X version
	# NB: this is exclusive with OPTION 1 above
	#
	#xcuserdata/**/*
	
	#     (requires option 2 above): Personal Schemes
	#
	#!xcuserdata/**/xcschemes/*
	
	####
	# XCode 4 workspaces - more detailed
	#
	# Workspaces are important! They are a core feature of Xcode - don't exclude them :)
	#
	# Workspace layout is quite spammy. For reference:
	#
	# /(root)/
	#   /(project-name).xcodeproj/
	#     project.pbxproj
	#     /project.xcworkspace/
	#       contents.xcworkspacedata
	#       /xcuserdata/
	#         /(your name)/xcuserdatad/
	#           UserInterfaceState.xcuserstate
	#     /xcsshareddata/
	#       /xcschemes/
	#         (shared scheme name).xcscheme
	#     /xcuserdata/
	#       /(your name)/xcuserdatad/
	#         (private scheme).xcscheme
	#         xcschememanagement.plist
	#
	#
	
	####
	# Xcode 4 - Deprecated classes
	#
	# Allegedly, if you manually "deprecate" your classes, they get moved here.
	#
	# We're using source-control, so this is a "feature" that we do not want!
	
	*.moved-aside
	
	####
	# UNKNOWN: recommended by others, but I can't discover what these files are
	#
	# ...none. Everything is now explained.

关于 `.gitignore` 我们还可以参考 [github官方的Objective-C.gitignore][3]。


## 如何在Xcode中使用git

在 `Xcode 5` 中，苹果的 Source Control 能很好的支持 git。具体如何使用，可以参考：[How To Use Git Source Control with Xcode in iOS 7](http://www.raywenderlich.com/51351/how-to-use-git-source-control-with-xcode-in-ios-7)。



## 一些场景

这里主要总结一些自己的使用场景，由于还算不上深度用户，欢迎大家批评指谬误之处。

### 开发某个项目的多个版本

开发某个项目的多个版本时，可以为不同的版本建立不同的分支来管理。

### 合并不同分支

现在我们有两个分支：b1 和 b2，我们当前在 b1 上。

	$ git branch
	master
	* b1
	b2

- 1）我们想把 b2 合并到 b1 里来，也就是 [Xcode][5] 里的 merge from，也就相当于在命令行直接合并b2: git merge b2。

		$ git merge b2

- 2）我们想把 b1 合并到 b2 里去，也就是 [Xcode][5] 里的 merge into，也就相当于在命令行先切换到 b2: git checkout b2(要记得所有更改都先 commit 了)；然后合并b1: git merge b1。

		$ git checkout b2
		$ git merge b1

<img src="/images/git-in-xcode/xcode-merge.png" alt="xcode-merge">

如果出现冲突，我们可以到 [Xcode][5] 中去 merge 了。

<img src="/images/git-in-xcode/xcode-merge-tool.png" alt="xcode-merge-tool">

### 遇到不能checkout的错误

想切换一下分支的时候遇到下面的错误：

	error: The following untracked working tree files would be overwritten by checkout:
	ProjectFolder.xcodeproj/project.xcworkspace/xcuserdata/myUserName.xcuserdatad/UserInterfaceState.xcuserstate
	Please move or remove them before you can switch branches.
	Aborting

遇到上面的错误，是因为项目中不应该被track的文件也被git给track了。

可以这样解决：

1）把上述文件加到 `.gitignore`，最好参考上文的 `.gitignore` 文件内容；

2）通过下面命令让git停止track相应的文件：

	git rm --cached ProjectFolder.xcodeproj/project.xcworkspace/xcuserdata/myUserName.xcuserdatad/UserInterfaceState.xcuserstate
	git commit -m "Removed file that shouldn't be tracked"

这之后git就会遵照 `.gitignore` 来忽略相应的文件了。



[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{page.title}})
[2]: http://samirchen.com/git-commands/
[3]: https://github.com/github/gitignore/blob/master/Objective-C.gitignore
[4]: http://zh.wikipedia.org/zh-cn/OS_X
[5]: http://zh.wikipedia.org/wiki/Xcode
[6]: http://zh.wikipedia.org/zh-cn/IOS