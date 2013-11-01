---
layout: post
title: git命令详解
description: 由于最近团队开发需大量使用git，在这里尽量详尽地介绍一下各种git命令，并总结出常用命令。
category: blog
---

##git简介
[Git][] 最初由 [Linus Torvalds][] 编写，用于 Linux 内核开发的版本控制工具。
[Git][] 与常用的版本控制工具 [CVS][]、[Subversion][] 等不同，它采用了分布式版本库的方式，不必服务器端软件支持，使源代码的发布和交流极其方便。

##本地（Local）

##1、初始化

###1.1、全局变量
	git config --global user.name "your_name"
	git config --global user.email "your@email.com"
	git config --list //列出git在该处找到的所有设置
	git config --global core.editor vim //设置你的缺省编辑器，Git在需要你输入一些消息时会使用该文本编辑器
	git config --global merge.tool diffmerge //设置merge工具

###1.2、初始化新版本库
	git init // 只会在根目录下创建 .git 文件夹

###1.3、设置忽略文件
设置每个人都想要忽略的文件：

- 1、在根目录创建一个 `.gitignore` 文件，在里面添加要忽略的文件或文件夹，一个一行；
- 2、将 `.gitignore` 文件加入版本库并提交

设置只有自己需要忽略的文件：

- 1、修改 `.git/info/exclude` 文件，可使用正则表达式

###1.4、添加新文件到版本库
	git add somefile.txt //添加单个文件
	git add *.txt //添加所有 txt 文件
	git add . //添加所有文件，包括子目录，但不包括空目录

###1.5、提交
	git commit -m "add all txt files"

##2、日常操作

###2.1、提交
	git commit -m "some msg" -a //提交所有修改
	git commit -m "add msg to readme.txt" readme.txt //提交单个文件
	git commit -C head -a --amend //不会产生新的提交历史记录，复用HEAD留言，增补提交，而不增加提交记录

###2.2、撤销修改
撤销尚未提交的修改（即没有commit的）：

	git checkout head readme.txt todo.txt //撤销1、2个文件
	git checkout head *.txt //撤销所有txt文件
	git checkout head . //撤销所有文件

撤销提交：

	//反转提交：
	git revert --no-commit head //反转提交，相当于提交最近一次提交的反操作
	
	//复位：
	git reset head //复位，取消暂存
	git reset head <filename> //复位，取消暂存
	git reset --hard head^ //复位到head之前的那个版本，不会在版本库中留下痕迹

###2.3、分支

	git branch //列出本地分支
	git branch -a //列出所有分支
	git branch <branchname> //基于当前分支末梢创建新分支
	git checkout <branchname> //检出分支
	git checkout -b <branchname> //基于当前分支末梢创建新分支并检出分支
	
	//基于某次提交、分支或标签创建新分支
	git branch emputy bfe57de0 //用来查看某个历史断面很方便
	git branch emputy2 emputy

	/*合并分支*/
	//普通合并
	git merge <branchname> //合并分支<branchname>到当前所在分支并提交，如果发生了冲突就不会自动提交，不想立即解决这些冲突的话，可以用 git checkout head . 撤销
	git merge --no-commit <branchname> //合并但不提交
	//压合合并
	git merge --squash <branchname> //压合合并后直接提交
	git merge --squash --no-commit <branchname> //当两个人合作开发一个新功能时，需要在一个分支提交多次，开发完成之后再压合成一次提交
	//拣选合并
	git cherry-pick --no-commit 5b62b6 //挑选某次提交合并但并不提交

	//重命名分支
	git branch -m <branchname> <newname> //不会覆盖已存在的同名分支
	git branch -M <branchname> <newname> //会覆盖已存在的同名分支

	//删除分支
	git branch -d new2 //如果分支没有被合并会删除失败
	git branch -D new2 //即使分支没有被合并也照删不误

###2.4、解决冲突
冲突很少时，直接编辑冲突的文件然后提交即可。

冲突比较复杂时，用 `git mergetool` 调用之前设定的merge工具来处理。

- 1、自动生成 BACKUP，BASE，LOCAL和REMOTE四个文件
- 2、调用设定好的merge工具
- 3、解决之后，关闭工具，BACKUP，BASE，LOCAL和REMOTE四个辅助文件会自动删除，但会同时生成一个 .orig 的文件来保存冲突前的现场。需手动删除这个文件
- 4、提交

###2.5、标签
	//创建标签
	git tag 1.0 //为当前分支最后一次提交创建标签，标签无法重命名
	git tag contacts_1.1 contacts //为contacts分支最近一次提交创建标签
	git tag 1.1 4e6861d5 //为某次历史提交创建标签

	git tag //显示标签列表

	git checkout 1.0 //检出标签，查看标签断面很方便

	git branch b1.1 1.1 //由标签创建分支
	git checkout -b b1.0 1.0

	git tag -d 1.0 //删除标签

###2.6、查看状态
	git status // 当前状态
	git log //历史记录
	git branch -v //每个分支最后的提交

###2.7、其他
	//导出版本库
	git archive --format=zip head>nb.zip
	git archive --format=zip --prefix=nb1.0/ head>nb.zip

##远端（Remote）

##1、初始化
###1.1、克隆版本库和添加远程版本库
1）当已经有一个远程版本库，只需要在本地克隆一份：

	git clone <giturl> //克隆，如：git clone https://github.com/me/test.git
	git clone <giturl> <dirname> //将远程库克隆到本地<dirname>目录下

克隆之后会自动添加四个config：

- remote.origin.fetch=+refs/heads/\*:refs/remotes/origin/\*
- remote.origin.url=https://github.com/me/test.git
- branch.master.remote=origin
- branch.master.merge=refs/heads/master

2）当在本地创建了一个工作目录，想把这个目录用git管理，并初始化到远程版本库，可以在远程服务器上创建一个目录，并把URL记录下来。在本地进入工作目录，然后执行：
	
	git init //初始化，对本地工作目录下的文件进行版本控制
	git remote add origin https://github.com/me/another_test.git //将本地工作目录放到远程服务器上。这样会添加URL地址为https://github.com/me/another_test.git，别名为origin的远程库

	//将本地的文件都推到远程库
	git add .
	git commit -am "init commit"
	git push origin master 

###1.2、别名
	git remote add <别名> <远程版本库URL> //添加远程版本库别名
	git remote rm <别名> //删除远程库的别名和相关分支

	git remote set-url --push <name> <newURL> //修改远程库

添加别名后会自动添加两个config：

- remote.origin.url=https://github.com/me/test.git
- remote.origin.fetch=+refs/heads/\*:refs/remotes/origin/\*

###1.3、创建一个无本地分支的库
	git init --bare //当需要一个公用的中央库时，非常适合把它建成bare库


##2、日常操作
###2.1、分支

	git branch -r //列出远程分支
	git branch -a //列出所有分支（本地和远程）
	git remote prune origin //删除远程库中已经不存在的分支
	
	git push origin --delete <branchName> //删除远程分支
	git push origin :<branchName> //推送一个空分支到远程分支，相当于删除远程分支

###2.2、从远程库获取

	git remote -v //查看远程仓库

	//获取但不合并
	git fetch <远程版本库> 
	//如：
	git fetch origin //origin是远程库的默认别名
	git fetch https://github.com/me/test.git

	//获取并合并到当前本地分支
	git pull //等价于git pull origin，origin是远程库的默认别名
	git pull https://github.com/me/test.git master //当不是从默认远程库获取时，需要指定获取哪个分支，这里就是指定获取master分支

	git pull <remoteName> <localBranchName> //拉取远程库到本地分支
	git pull origin test //拉取远程库到本地test分支
	
**推荐用下列方式从远程库获取代码：**

	git fetch origin master:tmp //从远程库origin的master分支获取版本库到本地tmp分支
	git diff tmp //比较本地tmp分支和当前所在分支（一般为本地master分支）
	git merge tmp //合并本地tmp分支到当前分支（一般为本地master分支）
	//如果这里遇到冲突，则参考“本地（Local） 2.4 解决冲突”来解决冲突

###2.3、推入远程库
	git push origin master //推入远程库，这里远程库默认别名origin，本地默认分支master

	git push origin test:master //将本地test分支推入远程库，并作为远程库的master分支
	git push origin test:test //将本地test分支推入远程库，并作为远程库的test分支

	

[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{page.title}})
[Git]: http://zh.wikipedia.org/wiki/Git "Git"
[Linus Torvalds]: http://zh.wikipedia.org/wiki/%E6%9E%97%E7%BA%B3%E6%96%AF%C2%B7%E6%89%98%E7%93%A6%E5%85%B9 "Linus Torvalds"
[Subversion]: http://zh.wikipedia.org/wiki/Subversion "Subversion"
[CVS]: http://zh.wikipedia.org/wiki/%E5%8D%94%E4%BD%9C%E7%89%88%E6%9C%AC%E7%B3%BB%E7%B5%B1 "CVS"