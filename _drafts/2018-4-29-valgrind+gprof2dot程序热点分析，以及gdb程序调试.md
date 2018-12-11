---
layout:     post
title:      编程秘术，化腐朽为神奇
subtitle:    "valgrind+gprod2dot程序程序调用和热点分析， gdb 程序调试"
date:       2018-4-29
author:     Mao Yanjun
header-img: img/post-myj-waves2Foam.png
catalog: true
tags:
    - Linux
    - C++
    - OpenFOAM
---
> 本部分可以编程秘术，强推。对于非计算机专业的人来讲这将会是你在程序学习和使用中的瑞士军刀，博主只恨相见恨晚，想想哪些搞不清楚函数调用的日子，不禁悲从中来，多少时间浪费在盲目猜测，又有多少时间在茫然不知所措。望君学习，有所收获。

## valgrind+gprod2dot程序程序调用和热点分析

程序热点分析：主要是为了搞清楚程序调用中的计算耗时部分在哪里。从而可以进行针对性的优化或者进行并行编程设置，可以说热点分析是并行编程必不可少的一部分。
关于valgrind 和gprod2dot的使用，在此放上几个博主参考链接，先移步学习。

[https://blog.csdn.net/sunmenggmail/article/details/10543483](https://blog.csdn.net/sunmenggmail/article/details/10543483) 推荐

[https://blog.csdn.net/liangpz521/article/details/20146767](https://blog.csdn.net/liangpz521/article/details/20146767 "Kcachegrind: ")

[https://blog.csdn.net/unix21/article/details/8772074](https://blog.csdn.net/unix21/article/details/8772074 "valgrind:")

链接很多，请自行百度先关内容。

### 安装：

博主用的Ubuntu系统，第一次竟然忘了最简单的方式：

```
      sudo apt-get install valgrind kcachegrind 
```

编译安装时，gcc版本问题无法通过进行编译，建议小白用Ubuntu。

### 配置

主要有下面两个重要配置;
* gprof2dot 按照上文参考链接1中所述，将可执行的gprof2dot.py 脚本拷贝到/usr/bin/中，需要root权限。方便了以后的使用。
* kcachegrind 可用apt install 安装

### 使用

本文以OpenFOAM中求解器为例

```
cd $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity/  //进入到算例目录
blockMesh //提前准备好算例，不同算例命令不同
valgrind --tool=callgrind icoFoam >log.icoFoam   //在valgrind 环境中利用callgrind来查看相关调用关系和耗时
gprof2dot.py -f callgrind callgrind.out.XXX |dot -Tpng -o report.png  
//可以将上述结果转化为关系图，支持图片格式png,eps,svg等等，
// gprof2dot.py也有相关参数 -n0 -e0 -s 等，可控制输出耗时的程度，复杂或者简化输出结果
```
调用结果如图所示：

ddd

关系图中所示，标红和标出的颜色即为系统瓶颈关键步骤，进一步，
图中第一个数为该步的调用总耗时百分比，括号中为执行该步本身的耗时百分比，第三个数是该函数被调用次数。关于关系图的进一步读图和清晰函数调用还需要进一步深入学习

对于上述结果，还可以用可视化的GUI工具 kcachegrind ，算例目录下执行即可：

```
kcachegrind callgrind.out.xxx
```
显示界面和关系图如图：

结果分析同上，看上去更为清晰简洁，但是失去了更多的函数调用信息。这方便可能是软件应用上需要进一步探索


## gdb 程序调试利器

> 强烈推出，我的救星。相信有了它，我就可以所向披靡的进入OpenFOAM世界了,也可以写更多博客了，哈哈哈。

### Debug 版OpenFOAM安装
gdb的相关使用：首先，进行程序调试OpenFOAM,必须要安装Debug版的OpenFOAM：
将：etc/bashrc中 WM_COMPILE_OPTION设置为Debug,其他设置不变：

```
#- Optimised, debug, profiling:
#    WM_COMPILE_OPTION = Opt | Debug | Prof
export WM_COMPILE_OPTION=Debug

```

### gdb 调试常用相关命令

```
cd $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity/  //进入到算例目录
blockMesh //提前准备好算例，不同算例命令不同

gdb icoFOAM
```
即可进入gdb调试模式

几个常用命令:下面两个链接都很有参考价值：

#### gdb常用命令和用法 

[https://www.cnblogs.com/TianFang/archive/2013/01/20/2868889.html](https://www.cnblogs.com/TianFang/archive/2013/01/20/2868889.html)

此部分最用要的就是搞清楚断点的设置，以及next 和 step 的区别 ``n`` 不进入子函数目录，直接运行一步，``s`` 一步步步进，可进入子函数。

#### gdb调试动态链接库

[https://blog.csdn.net/yasi_xi/article/details/18552871](https://blog.csdn.net/yasi_xi/article/details/18552871)

断点设置中可以跨到动态链接库或者其他**.C文件中，主要由以下几种设置方式，亲测OpenFOAM可以已经库文件加入搜索路径：

```
b sixDoFRigidBodyMotionSolver.C:168 //此种方式直接跨文件设置断点，知道程序调用规则可以查看相关变量值，不知道函数调用关系可以通过猜测，设置断点看是否调用了此源文件。
b Foam::sixDoFRigidBodyMotionSolver::solve //函数名要写全，不需要（）
```

#### 断点的保存：

有的时候发现运行起来断点数设置不对，可能需要对已经设置的断点进行保存，方便下一次使用：
> 1. 保存断点
> 
>     先用info b 查看一下目前设置的断点，使用save breakpoint命令保存到指定的文件，这里我使用了和进程名字后面加bp后缀，你可以按你的喜好取名字。
>     save breakpoint fig8.3.bp
> 2. 读取断点
> 
>     注意，在gdb已经加载进程的过程中，是不能够读取断点文件的，必须在gdb加载文件的命令中指定断点文件，具体就是使用-x参数。例如，我需要调试fig8.3这个文件，指定刚才保存的断点文件fig8.3.bp。

```
info b
save breakpoint ***.*.bp
gdb icoFoam -x ***.*.bp
```

然后还可以继续添加断点，继续进行相关调试

[https://blog.csdn.net/u011068702/article/details/53931648](https://blog.csdn.net/u011068702/article/details/53931648)

#### 将GDB中需要的调试信息输出到文件

将调试信息进行保存，便于调试后的整体分析和程序分析。

```
 (gdb) set logging file <文件名>
(gdb) set logging on
 (gdb) thread apply all bt
(gdb) set logging off
 (gdb) quit
```

详细说明：

1、# (gdb) set logging file <文件名>

设置输出的文件名称

2、# (gdb) set logging on

输入这个命令后，此后的调试信息将输出到指定文件

3、# (gdb) thread apply all bt

打印说有线程栈信息

4、# (gdb) set logging off

输入这个命令，关闭到指定文件的输出

[http://blog.chinaunix.net/uid-20757375-id-3018189.html](http://blog.chinaunix.net/uid-20757375-id-3018189.html)

> **更多问题和建议可以在下方评论区留言讨论，或者发我邮箱，邮箱在ABOUT中。原创帖转载请留言并注明出处**










