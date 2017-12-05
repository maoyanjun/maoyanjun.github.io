---
layout:     post
title:      SPHERA
subtitle:    SPHERA 在linux上的安装和使用
date:       2017-12-05
author:     Mao Yanjun
header-img: img/post-myj-sphera.png
catalog: true
tags:
    - CFD
    - SPH
    - Ocean Engineering
---

> “🙉🙉🙉 ”

# SPHERA的编译安装与尝试


> SPHERA v.8.0 (RSE SpA) is SPH research and free software. So far (2015), SPHERA has been characterized by two alternative boundary treatment schemes (based on either volume integrals or discrete surface elements), a 2D erosion criterion, a scheme for body transport in free surface flows. SPHERA has represented several types of floods (with transport of solid bodies and granular material) and sloshing tanks. SPHERA is published and developed as FOSS (Free/Libre and Open Source Software) on GitHub at https://github.com/AndreaAmicarelliRSE/SPHERA.

本文主要讲述如何在linux上编译安装SPHERA
## 代码获取：
Git: ```git clone git@github.com:AndreaAmicarelliRSE/SPHERA.git```
SVN:```svn https://github.com/AndreaAmicarelliRSE/SPHERA.git```

注：下载链接经常失效，可自行到GitHub上搜索SPH，查看相关开源代码
## 编译安装
编译的核心是使用Makefile文件进行文件的编译和安装，makefile文件如下，编译安装前，建议简单学习一下makefile文件的文件结构和使用方法，本程序是基于Fortran语言进行编写的，需要用到gfortran 或者ifort 编译器，当然，makefile的方法是在linux下通用的编译大型程序的方式。可用于C语言和C++语言等程序的编写。因此有必要学习一下。
参考学习网站：[Makefile 学习](http://wiki.ubuntu.org.cn/跟我一起写Makefile:MakeFile介绍)
makefile如下，基本原理是定义几个环境变量，然后利用环境变量和通配符来进行批量的文件编译和链接。
```# Variables to be updated
VERSION = 8_0_AA
#    bin, debug 用于存放可执行文件的位置，用opt模式的存于bin中，debug模式存放于debug文件下
EXECUTION = bin
#    -O1 (bin), -g -O0 -fbacktrace -C (gfortran debug),
#    -g -O0 -traceback -C -check bounds -check noarg_temp_created -debug all (ifort debug)
#编译的标签，
COMPILATION_FLAGS = -O1
#    gfortran, ifort 默认编译器选择为gfortran 或者ifort，笔者采用gfortran进行了相关编译工作。
COMPILER = gfortran
#    -fopenmp, -qopenmp (formerly -openmp) 并行模式的定义
OMP_FLAG = -fopenmp
#存放可执行文件的目录，在src的上一级存在或者新建一个bin 文件夹。
EXE_DIR = ../$(EXECUTION)/
#通过SOURCES列出了所有待编译程序的相对路径
SOURCES = \
./Modules/Static_allocation_module.f90 \
./Modules/Hybrid_allocation_module.f90 \
./Modules/Dynamic_allocation_module.f90 \
./Modules/I_O_diagnostic_module.f90 \
./Modules/I_O_ENG_module.f90 \
./Modules/I_O_file_module.f90 \
./Modules/I_O_ITA_module.f90 \
./Modules/I_O_language_module.f90 \
./Modules/SA_SPH_module.f90 \
./Modules/Time_module.f90 \
$(wildcard ./BC/*.f90) \
$(wildcard ./BE_mass/*.f90) \
$(wildcard ./BE_momentum/*.f90) \
$(wildcard ./Body_Transport/*.f90) \
$(wildcard ./Constitutive_Equation/*.f90) \
$(wildcard ./DB_SPH/*.f90) \
$(wildcard ./Erosion_Criterion/*.f90) \
$(wildcard ./Geometry/*.f90) \
$(wildcard ./IC/*.f90) \
$(wildcard ./Interface_dispersion/*.f90) \
./Main_algorithm/Gest_Dealloc.f90 \
./Main_algorithm/Gest_Trans.f90 \
./Main_algorithm/Loop_Irre_2D.f90 \
./Main_algorithm/Loop_Irre_3D.f90 \
$(wildcard ./Neighbouring_Search/*.f90) \
$(wildcard ./Post_processing/*.f90) \
$(wildcard ./Pre_processing/*.f90) \
$(wildcard ./SA_SPH/*.f90) \
$(wildcard ./Strings/*.f90) \
$(wildcard ./Time_Integration/*.f90)
# Other variables
#主文件的位置sphera，程序主体
MAIN_FILE_ROOT = ./Main_algorithm/sphera
#最后编译出来的可执行文件的名字
CODE = $(EXE_DIR)SPHERA_v_$(VERSION)_$(COMPILER)_$(EXECUTION)
#定义所有相关的源文件和.o文件的变量
OBJECTS = $(patsubst %.f90,%.o,$(SOURCES))
# Targets 此部分是makefile的核心部分，表达方式是makefile特有方式，具体参考学习。
#    Correct sequence for compile+link:
#       make touch
#       make
all: compile link remove
#    "compiling" cannot be launched together with "touching" as ".o" files
#    would not be available yet
compile: $(SOURCES)
%.f90: %.o
       $(COMPILER) $@ -o $< $(OMP_FLAG) $(COMPILATION_FLAGS) -c
link: $(OBJECTS)
       touch $(CODE).x
       $(COMPILER) $(MAIN_FILE_ROOT).f90 $^ -o $(CODE).x $(OMP_FLAG) $(COMPILATION_FLAGS)
       rm -f $(MAIN_FILE_ROOT).o
remove: $(OBJECTS)
       rm -f $^
       rm -f *.mod
touch:
       touch $(OBJECTS)
clean:
       rm -f $(CODE).x
#*************************************#
```
此处强调一下，执行顺序
* make touch
* make
* make touch 作为单独的命令，用于产生编译所需的前提可执行文件，相当于编译预处理。
* make 用于进行程序的编译和链接。

Compile 将 make touch 创建的 *.o文件编译成 *.mod 模块文件
Link 将相关模块 *.mod 进行与主函数的链接，形成可执行文件*.x
Remove 清除相关中间文件
Clean 清除最终生成的可执行文件 CODE.x
注：可执行文件在linux下后缀名没有太大意义，此处定义为 *.x
**编译过程出现的问题**：
1. file not recognized: File truncated
此问题出现原因是可执行文件SPHERA_v_7_2_gfortran_run.x 不能有空格，而此时，在定义 EXECUTION = run 时容易引入空格，导致文件名断裂SPHERA_v_7_2_gfortran  _run.x，从而出现以上错误。
2. linker input file unused because linking not done
此问题是因为make complie 出现了问题，
%.f90: %.o
         $(COMPILER) $@ -o $< $(OMP_FLAG) $(COMPILATION_FLAGS) –c
标红部分，在7.2版本中存在写反的情况，导致编译出错。此时为什么出错，请自行学习，参考8.0版本，修改后即可成功编译
#%.f90: %.o
%.o: %.f90
       $(COMPILER) $< -o $@ $(OMP_FLAG) $(COMPILATION_FLAGS) –c
3. 经过尝试，8.0版本可以顺利运行，但是所带的input 算例中在*.inp deck 读取的时候， RGBColor 处出错，可能是版本兼容问题。也可能是程序编译过程存在问题。
经尝试7.2版本可以正常运行，计算结果。
最后，已经编译好的可执行文件8.0和7.2版本的代码可以从下面获取：
```git clone git@github.com:maoyanjundut/SPH.git```
       
### 算例运行
算例目录下需要有一下三个文件
├── dam_break_2D_demo.inp #基本参数输入文件
├── SPHERA_v_8_0_AA_gfortran_bin.x #可执行文件
└── surface_mesh_list.inp #相关定义文件，默认存在，无需更改
在算例目录下输入：
```./ SPHERA_v_8_0_AA_gfortran_bin.x dam_break_2D_demo```
注意：是dam_break_2D_demo，不带后缀。
 
### 后处理
后处理采用paraview 或者tecplot。 注意应同时导入 *.vtk 和*.vtu 文件，其中vtk文件只包含算例几何文件，vtu文件是相关计算结果
![paraview postprocess](https://i.imgur.com/5YfZt05.jpg)






