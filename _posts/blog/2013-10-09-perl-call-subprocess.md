---
layout: post
title: Perl中调用子进程的常用方法
description: 介绍3种在Perl中调用子进程的常用方法。
category: blog
---

##场景

在Unix-like环境下，通常会使用Perl脚本来做一些文本处理、数据处理、系统命令测试之类的工作。在这些工作中经常会需要我们调用别人的代码或程序来完成某一部分工作，这就涉及到如何在Perl中调用子进程的问题。

比如，最近我们搭建好了万兆网系统，需要用[iperf][]测试则个网络的实际传输速度，在这个过程中，需要测试在read/write buffer大小不同的情况下的传输速度，我们对这个buffer大小可能有上百种取值点，比如我们会测4KB、8KB、12KB等等情况下的传输速度，如果不用脚本去实现自动化测试，我们就得一条一条地去自己敲类似下面的命令：

	iperf -c 192.168.0.1 -t 300 -l 4k
	iperf -c 192.168.0.1 -t 300 -l 8k
	iperf -c 192.168.0.1 -t 300 -l 12k
	......

如果使用脚本的话，我们在脚本中让Perl去一遍一遍地修改设置read/write buffer的大小，并调用对应的测试程序命令即可。我们所要做的就是启动一次脚本，接着等结果就好了。这个过程中我们就要用Perl去调用测试程序了。

##Perl中调用子进程的方式

###使用`system`函数

这样调用子进程可以得到一个返回值，这个返回值右移`8`位后与在Shell中调用这个程序得到的返回值一致。
几个例子：

1.1 我们在Perl脚本中调用`date`命令：

    #!/usr/bin/perl
    use strict;
    use warnings;
    sytem "date";

执行：

	[cx@rh Perl]$ perl test.pl
	Wed Mar 27 06:45:07 CST 2013

1.2 得到`system`调用函数的返回值：

	#!/usr/bin/perl
	use strict;
	use warnings;
	my $i = system "service httpd status";
	print $i . "\n";

执行：

	[cx@rh Perl]$ perl test.pl
	httpd dead but subsys locked
	512

我们发现这里的返回值是`512`，是有问题的。因为我们自己在Shell下执行`service httpd status`这条命令得到的返回值是`2`：

	[cx@rh Perl]$ service httpd status
	httpd dead but subsys locked
	[cx@rh Perl]$ echo $?
	2

问题是这样的，在Perl中使用`system`调用函数得到的返回值应该右移`8`位才等于在Shell中调用这条命令得到的返回值。所以上面的那个脚本应该改为：

	#!/usr/bin/perl
	use strict;
	use warnings;
	my $i = system "service httpd status";
	$i = $i >> 8;
	print $i . "\n";

这样结果就与Shell中调用所得到的返回值一致了。

###使用`exec`函数

这样调用子进程后，Perl进程会结束掉，然后只运行这个被调用的子进程。这种方式可以在只想把Perl脚本当做一个启动其他进程的启动器时用到。
几个例子：

2.1 

	#!/usr/bin/perl
	use strict;
	use warnings;
	print "Begin\n";
	exec "service httpd status";
	print "End\n";

执行：

	[cx@rh Perl]$ perl test.pl
	Statement unlikely to be reached at test.pl line 8.
	(Maybe you meant system() when you said exec()?)
	Begin
	httpd dead but subsys locked

可以看到，结果并没有打印出`End`，而且Perl还给了我们一个提示，意义很明显。Perl进程在执行完`exec`函数后就退出了。

###使用`\` \``(反引号)

这样调用进程可以捕捉到调用该进程的输出文本，这个输出文本与直接在Shell下调用这个进程一致。这个是很有用的，我们有很多程序都会输出文本给用户以反馈，我们可以在Perl中捕捉这些信息，对其进行处理分析，这个在自动化测试、数据处理等工作中搭配上Perl的正则表达式等功能尤其有用。
几个例子：

3.1 我们用变量`$msg`来存储命令`service httpd status`的输出，再把它打印出来：

	#!/usr/bin/perl
	use strict;
	use warnings;
	my $msg = \`service httpd status\`;
	print $msg;

执行：

	[cx@rh Perl]$ perl test.pl
	httpd dead but subsys locked


[iperf]: http://iperf.sourceforge.net/ "iperf"