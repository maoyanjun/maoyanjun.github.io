---
layout:     post
title:      waves2FOAM的安装与使用
subtitle:    ""
date:       2018-1-10
author:     Mao Yanjun
header-img: img/post-myj-waves2Foam.png
catalog: true
tags:
    - OpenFOAM
    - waves2Foam
    - CFD
    - NWT
---

> “🙉🙉🙉 ”

> **本帖为新手帖，学神可以忽略，或者提供补充意见。会持续更新补充，新手所写，理论不严谨，代码不简洁，逻辑不清晰，仅供学习记录参考**

# 新手建议，此处是重点
1. 阅读英文资料，网站，论坛，特别是针对于研究人员，这是使用OpenFOAM这样程序的基本要求 
2. 对于OpenFOAM的学习建议，建议Linux系统，，软件一定要自己安装几遍，多去折腾一些，才会有更多的感悟和收获，此外，Linux系统建立在开源框架下，有更多优秀的开源程序是基于该系统下的，方便后期的程序使用。
3. 熟悉Linux系统的使用，尤其是环境变量的基本意义，就像是全局变量一样，对于软件安装和使用过程大有裨益，回头看来，这个概念其实在windows下也同样重要，只是被我们忽略了而已，在很多破解或者编译软件的过程过程中都会遇到这个概念的。
4. 戒骄戒躁

# waves2Foam的安装与使用
关于wavesFoam的安装与使用，[http://http://openfoamwiki.net/index.php/Contrib/waves2Foam](http://http://openfoamwiki.net/index.php/Contrib/waves2Foam "openfoamwiki"),上面已经做了详细的介绍，其中主要内容都迁移到了wave2Foam Manual 上去了。因此对于使用waves2Foam的人员，建议通读manualwaves2Foam.pdf。其中安装部分已经把基本安装流程和所需要的依赖都说明白了，先移步去尝试尝试可好？这里主要是点出安装过程中对于新手常遇到的几个小坑。
# 支持版本
手册中已经写明了支持的版本，亲测2.x ,3.x系列几乎安装没有太大问题，目前阶段OF1706版本暂时编译不了。

# 相关依赖
一定要把manualwaves2Foam中的相关依赖都下载和编译安装好之后再进行waves2Foam的安装。依赖的相关安装方法自行百度。这里面有个很容易出的问题，就是ThirdParrty里面，关于OceanWave3D-Fortran90的下载，与安装时采用的git clone.
``git clone https://github.com/boTerpPaulsen/OceanWave3D-Fortran90.git  ``,详细见``ThirdParty/Allwmake``如下：

```
#!/bin/bash

## The script has been modified on 2015.09.08 to reflect the fact that
#  the svn-repository for OceanWave3D has been discontinued.

sourceLib="lib"
settings="settings"

if [ ! -d "$sourceLib" ]
then
    mkdir $sourceLib
fi


echo "====================================="
echo "    COMPILE LAPACK-3.3.1"
echo "====================================="

compileFlag=0

if [ ! -f "$sourceLib/libblas.a" ]
then
    compileFlag=1;
fi

if [ ! -f "$sourceLib/liblapack_gfortran.a" ]
then
    compileFlag=1;
fi

if [ ! -f "$sourceLib/libtmglib_gfortran.a" ]
then
    compileFlag=1;
fi

## COMPILE LAPACK
if [ "$compileFlag" == "1" ]
then
    # Unpack lapack-3.3.1
    tar -xzf lapack-3.3.1.tgz

    # Copy the necessary settings
    cp $settings/lapackSettings/* lapack-3.3.1/.

    # Compile lapack
    cd lapack-3.3.1
    make clean
    make all
    cd ..
else
    echo
    echo "lapack-3.3.1 has already been compiled"
    echo
fi


echo "====================================="
echo "    COMPILE SPARSKIT2"
echo "====================================="

compileFlag=0

if [ ! -f "$sourceLib/libskit_gfortran.a" ]
then
    compileFlag=1;
fi

## COMPILE SPARSKIT2
if [ "$compileFlag" == "1" ]
then
    # Unpack SPARSKIT2
    tar -xzf SPARSKIT2.tar.gz

    # Copy the necessary settings
    cp $settings/sparseSettings/makefile SPARSKIT2/.

    # Compile SPARSKIT2
    cd SPARSKIT2
    make clean
    make lib
    mv libskit_gfortran.a ../lib/.
    cd ..
else
    echo
    echo "SPARSKIT2 has already been compiled"
    echo
fi


echo "====================================="
echo "    COMPILE OCEANWAVE3D"
echo "====================================="

# Path for the git-repository
ocw="OceanWave3D-Fortran90"

# Create output directory for OceanWave3D
if [ ! -d "bin" ]
then
    mkdir bin
fi

# Check-out the git-repository
if [ ! -d "$ocw" ]
then
    echo ""
    echo "Cloning the OceanWave3D git repository ..."
    git clone https://github.com/boTerpPaulsen/OceanWave3D-Fortran90.git  

    # Copy the compilation settings for OCW3D
    cp $settings/oceanWave3DSettings/common.mk $ocw/.
    cp $settings/oceanWave3DSettings/makefile $ocw/.
else
    echo ""
    echo "Pull changes from the OceanWave3D git repository ..."
    cd $ocw
    
    # Stash the local changes on the settings
   # git stash

    # Get an update of the git and report the result
  #  output=`git pull`
    
    # Check whether the git was already up to date
    if [[ $output == "Already up-to-date." ]]
    then
        compileFlag=0
    else
        compileFlag=1
    fi

    # Re-apply the compilation settings
    git stash pop

    cd $OLDPWD
fi

# Check for the correct environmental variables
if [ -z "$WAVES_LIBBIN" ]
then
    echo ""
    echo "Set the environmental variables for waves2Foam"
    echo "Exiting compilation process"
    echo ""
    exit 1
fi

# Check whether the executable has been deleted
if [ ! -f "$WAVES_APPBIN/OceanWave3D" ]
then
    compileFlag=1
fi

# Check whether the library has been deleted
if [ ! -f "$WAVES_LIBBIN/libOceanWave3D.so" ]
then
    compileFlag=1
fi

if [ "$compileFlag" == "1" ]
then
    # Temporary build directory
    mkdir build

    # Compile OceanWave3D input an executable and a dynamic library
    cd $ocw

    make clean
    make Release
    make shared

    cd $OLDPWD

    # Clean up
    rm -rf build
else
    echo
    echo "OceanWave3D has already been compiled"
    echo
fi

pwd

echo "====================================="
echo "    COMPILE FENTON4FOAM"
echo "====================================="

fentonName="fenton4Foam"

if [ ! -f "$WAVES_APPBIN/$fentonName" ]
then
    echo
    echo "Compiling fenton4Foam"
    echo
    cd $fentonName
    gfortran -o $WAVES_APPBIN/$fentonName $fentonName.f fft.f
    cd $OLDPWD
else
    echo
    echo "fenton4Foam has already been compiled"
    echo
fi

exit 0
```

这里面关键部分是这部分容易出问题，如果OceanWave3D编译出现问题，请参考下面这段的介绍：

```
echo "====================================="
echo "    COMPILE OCEANWAVE3D"
echo "====================================="

# Path for the git-repository
ocw="OceanWave3D-Fortran90"

# Create output directory for OceanWave3D
if [ ! -d "bin" ]
then
    mkdir bin
fi

# Check-out the git-repository
if [ ! -d "$ocw" ]
then
    echo ""
    echo "Cloning the OceanWave3D git repository ..."
    git clone https://github.com/boTerpPaulsen/OceanWave3D-Fortran90.git  

    # Copy the compilation settings for OCW3D
    cp $settings/oceanWave3DSettings/common.mk $ocw/.
    cp $settings/oceanWave3DSettings/makefile $ocw/.
else
    echo ""
    echo "Pull changes from the OceanWave3D git repository ..."
    cd $ocw
    
    # Stash the local changes on the settings
   # git stash

    # Get an update of the git and report the result
  #  output=`git pull`
    
    # Check whether the git was already up to date
    if [[ $output == "Already up-to-date." ]]
    then
        compileFlag=0
    else
        compileFlag=1
    fi

    # Re-apply the compilation settings
    git stash pop

    cd $OLDPWD
fi

# Check for the correct environmental variables
if [ -z "$WAVES_LIBBIN" ]
then
    echo ""
    echo "Set the environmental variables for waves2Foam"
    echo "Exiting compilation process"
    echo ""
    exit 1
fi

# Check whether the executable has been deleted
if [ ! -f "$WAVES_APPBIN/OceanWave3D" ]
then
    compileFlag=1
fi

# Check whether the library has been deleted
if [ ! -f "$WAVES_LIBBIN/libOceanWave3D.so" ]
then
    compileFlag=1
fi

if [ "$compileFlag" == "1" ]
then
    # Temporary build directory
    mkdir build

    # Compile OceanWave3D input an executable and a dynamic library
    cd $ocw

    make clean
    make Release
    make shared

    cd $OLDPWD

    # Clean up
    rm -rf build
else
    echo
    echo "OceanWave3D has already been compiled"
    echo
fi

```

这一部分由于网络问题或者git 库可能迁移，会导致无法下载OceanWave3D,从而引发一系列问题。导致编译失败。此部分可以通过阅读make 文件的操作，进行手动解决OceanWave3D的编译，主要进行了以下操作：

* 下载OceanWave3D 到ThirdParty/目录下，可手动从github上搜索下载。
*  ``cp $settings/oceanWave3DSettings/common.mk $ocw/.``
*  ``cp $settings/oceanWave3DSettings/makefile $ocw/.``

完成这三步，就相当于手动解决了OCeanWave3D的下载问题，然后可以运行Allmake进行编译。或者自行阅读后续过程手动编译OceanWave3D.

# 环境变量设置

这里按照手册要求，安装路径是 ``$FOAM_RUN/../applications/utilities``，这里使用了OF的环境变量FOAM_RUN,新手一定要将源代码放入此路径中，因为默认的环境变量$WAVES_DIR是这个路径，当然，进一步如果了解到了这个问题，你可以把waves2Foam安装到任意位置。环境变量设置文件位于``\bin\bashrc.org``下面，此路径中定义了所有waves2Foam的环境变量。下面是文件示例：

```
#!/bin/bash

### SETTING A FLAG
if [ -z "$WAVES_DIR" ]
then
    printVariables=1
fi

# If any string is parsed, do no print
if [ -n "$1" ]
then
    printVariables=0
fi

### USER DEFINED ENVIRONMENTAL VARIABLES
export WAVES_DIR=$WM_PROJECT_USER_DIR/applications/utilities/waves2Foam
export WAVES_APPBIN=$FOAM_USER_APPBIN
export WAVES_LIBBIN=$FOAM_USER_LIBBIN

export WAVES_GSL_INCLUDE=/usr/include
export WAVES_GSL_LIB=/usr/lib64

### OTHER STATIC ENVIRONMENTAL VARIABLES
## Version number - first old numbering system for foam-extend, OpenFoam and up to OpenFoam-v3.0+
# The "-0" is for zero-padding the version number, to accept differences between e.g. 2.1 and 2.1.1
if [ `uname` = "Darwin" ]
then
    version=`echo $WM_PROJECT_VERSION"-0" | sed -e 's/\.x/-0/' -e 's/\./\'$'\n/g' -e 's/[v+-]/\'$'\n/g' | grep "[0-9]" | head -3 | tr -d '\n'`
else
    version=`echo $WM_PROJECT_VERSION"-0" | sed -e 's/\.x/-0/' -e 's/\./\n/g' -e 's/[v+-]/\n/g' | grep "[0-9]" | head -3 | tr -d '\n'`
fi

# Version number for the OpenFoam-v1606+ version numbering (no dots, etc)
flag=`echo $WM_PROJECT_VERSION | grep "^v[0-9]*+"`
echo $flag
if [ -n "$flag" ]
then
    if [ `uname` = "Darwin" ]
    then
        version=`echo $WM_PROJECT_VERSION | sed -e 's/[v+-]/\'$'\n/g' | tr -d '\n'`
    else
        version=`echo $WM_PROJECT_VERSION | sed -e 's/[v+-]/\n/g' | tr -d '\n'`
    fi
fi

export WM_PROJECT_VERSION_NUMBER=$version

## Easy navigation
export WAVES_SRC=$WAVES_DIR/src
export WAVES_TUT=$WAVES_DIR/tutorials
export WAVES_SOL=$WAVES_DIR/applications/solvers/solvers$WM_PROJECT_VERSION_NUMBER
export WAVES_UTIL=$WAVES_DIR/applications/utilities
export WAVES_PRE=$WAVES_UTIL/preProcessing
export WAVES_POST=$WAVES_UTIL/postProcessing

## Is this a D.D.x version of OpenFoam. Needed to distinguish 2.2.0 and 2.2.x
truncatedVersion=${WM_PROJECT_VERSION%.x}

if [ "$WM_PROJECT_VERSION" == "$truncatedVersion" ]
then
    xVersion=0
else
    xVersion=1
fi

export WAVES_XVERSION=$xVersion

## Extend branch or not
EXTBRANCH=`echo $WM_PROJECT_VERSION | grep 'dev\|ext'`

if [ -z $EXTBRANCH ]
then
    EXTBRANCH=0
else
    EXTBRANCH=1
fi

export EXTBRANCH

## Is it the new foam-extend class?
FOAMEXTENDPROJECT=0

if [ $WM_PROJECT == "foam" ]
then
    FOAMEXTENDPROJECT=1
fi

export FOAMEXTENDPROJECT

if [ "$FOAMEXTENDPROJECT" == "1" ]
then
    EXTBRANCH=$FOAMEXTENDPROJECT
    export EXTBRANCH
    export WAVES_SOL=${WAVES_SOL}"_EXT"
fi

## Is it the OpenFoam+ version?
OFPLUSBRANCH=`echo $WM_PROJECT_VERSION | grep '+'`

if [ -z $OFPLUSBRANCH ]
then
    OFPLUSBRANCH=0
else
    OFPLUSBRANCH=1
    export WAVES_SOL=${WAVES_SOL}"_PLUS"
fi

export OFPLUSBRANCH

### PRINT INFORMATION
if [ "$printVariables" == "1" ]
then
    echo ""
    echo "====================================="
    echo "    ENVIRONMENTAL VARIABLES"
    echo "====================================="
    env | grep "WAVES\|EXTBRANCH\|WM_PROJECT_VERSION_NUMBER\|FOAMEXTEND\|OFPLUSBRANCH" | sort
    echo ""
fi
```

上面是整个的bashrc文件，我们需要关心的主要是一下部分，#号表示注释内容

```
### USER DEFINED ENVIRONMENTAL VARIABLES
export WAVES_DIR=$WM_PROJECT_USER_DIR/applications/utilities/waves2Foam #此路径是waves2Foam源文件所处位置，与用户手册中的下载源码前cd到的位置相对应，可以更改，新手建议不改。
export WAVES_APPBIN=$FOAM_USER_APPBIN #此路径是编译安装后的求解器和工具的可执行文件的放置位置，编译成功后可以到相关路径下查看可执行文件，前提是一定要source 了OF的环境变量。
export WAVES_LIBBIN=$FOAM_USER_LIBBIN #同理如上文，高阶用户可以定义自己存放的位置，$FOAM_USER_LIBBIN以及是用户自己的存放目录，可以通过 cd $FOAM_USER_LIBBIN 切换到所代表的路径

export WAVES_GSL_INCLUDE=/usr/include #这两个默认就好
export WAVES_GSL_LIB=/usr/lib64
```

还有一块叫做Easy navigation，此部分的使用方法是将该bashrc.org的路径到自己的配置文件中``./.bashrc``中去，参考多版本OF控制的原理，在此不想详述。

```
## Easy navigation
export WAVES_SRC=$WAVES_DIR/src
export WAVES_TUT=$WAVES_DIR/tutorials
export WAVES_SOL=$WAVES_DIR/applications/solvers/solvers$WM_PROJECT_VERSION_NUMBER
export WAVES_UTIL=$WAVES_DIR/applications/utilities
export WAVES_PRE=$WAVES_UTIL/preProcessing
export WAVES_POST=$WAVES_UTIL/postProcessing
```

剩下的环境变量部分主要是进行的版本的区别和不同进行了一定的修改和设置。

**补充：使用默认的路径编译好可执行文件后，后续如果自己改动求解器编译时路径尽量不要使用waves2Foam提供了的环境变量，最好使用绝对路径，虽然麻烦了一些但是不会出现太多的环境变量问题，或者如上文所述，记得将其添加到自己的配置文件中，每次开启终端即可自动source .bashrc的话也可以**

# 算例运行

默认算例可以通过运行Allrun命令来运行算例，此处可以通过阅读Allrun来了解waves2Foam算例运行过程中的执行的命令，可以手动按顺序执行。此处一个小坑，算例换到其他位置后无法运行Allrun。请看代码注释部分

```
#!/bin/bash

# Source tutorial run functions
. $WM_PROJECT_DIR/bin/tools/RunFunctions

#此部分是这个大坑的所在，算例设置部分调用了一个prepareCase，但是采用了相对路径，这导致算例只能在它的算例目录下，
#其他位置就会出现问题，无法准备算例的相关设置，因此此处如果使用请自行修改prepareCase.sh的执行位置，
#或者将此部分内容根据自己的需要写入到Allrun文件中，更为方便。
exec="../../../bin/prepareCase.sh"

if [ -x "$exec" ]
then
    . $exec
else
    echo "The file $WAVES_DIR/bin/prepareCase.sh is not executable."
    echo "Make the file executable before continuing."
    echo
    echo "Exiting tutorial case."
    exit 1
fi

# Set application name
application="waveFoam"

# Create the computational mesh
runApplication blockMesh

# Create the wave probes
runApplication waveGaugesNProbes

# Compute the wave parameters
runApplication setWaveParameters

# Set the wave field
runApplication setWaveField

# Run the application
runApplication $application

# To a post-processing analysis
runApplication postProcessWaves2Foam
```

# waveDyMFoam的修改
部分版本中waveDyMFoam需要自己通过interDyMFoam进行修改，此处的主要问题就是需要理解Make文件夹中的两个文件file和option

file:**.C文件文件名，和可执行文件的存放位置，注意此处最好手动写绝对路径。

```
waveFoam.C

EXE = $(WAVES_APPBIN)/waveFoam
```
options:源文件需要引用的相关代码和库文件，编译时，时常出现缺失某些文件的情况，主要是此处路径的问题，可以在此处添加，include进来，强调一点，一定要注意路径书写的正确性，不要换行书写，注意结尾的 \。
小写 -i include某个文件，大写 -I include某个文件夹下所有，同理 -l & -L

```
EXE_INC = \
    -I$(LIB_SRC)/transportModels/twoPhaseMixture/lnInclude \
    -I$(LIB_SRC)/transportModels \
    -I$(LIB_SRC)/transportModels/incompressible/lnInclude \
    -I$(LIB_SRC)/transportModels/interfaceProperties/lnInclude \
    -I$(LIB_SRC)/turbulenceModels/incompressible/turbulenceModel \
    -I$(LIB_SRC)/transportModels/immiscibleIncompressibleTwoPhaseMixture/lnInclude \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/fvOptions/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude \
    -I$(LIB_SRC)/sampling/lnInclude \
    -DOFVERSION=240 \
    -DEXTBRANCH=0 \
    -DXVERSION=$(WAVES_XVERSION) \
    -DOFPLUSBRANCH=0 \
    -I$(WAVES_SRC)/waves2Foam/lnInclude \
    -I$(WAVES_SRC)/waves2FoamSampling/lnInclude \
    -I$(WAVES_GSL_INCLUDE)

EXE_LIBS = \
    -limmiscibleIncompressibleTwoPhaseMixture \
    -lincompressibleTurbulenceModel \
    -lincompressibleRASModels \
    -lincompressibleLESModels \
    -lfiniteVolume \
    -lfvOptions \
    -lmeshTools \
    -lsampling \
    -L$(WAVES_LIBBIN) \
    -lwaves2Foam \
    -lwaves2FoamSampling \
    -L$(WAVES_GSL_LIB) \
    -lgsl \
    -lgslcblas
```

# 杂项
对于新手，经常会遇到的``命令不存在``的错误提示，思考了一下，大概有一下几种可能参考：
1. 命令输入不对 可能性：30%  解决方法：核对命令，注意大小写
2. 软件安装失败，命令本身不存在 可能性：30% 解决方法：重新检查软件安装过程，是否出错
3. 部分命令因为版本或者特殊原因安装失败，此时可以根据自己的命令到存放可执行文件的文件夹，默认``$FOAM_USER_APPIN``和``$FOAM_USER_LIBBIN``中查看是否存在 可能性：20% 解决方法：如果不存在，则找到对应的源程序，重新尝试wmake进行编译，或者重装整个程序。
4. 编译的可执行文件没有加入到Linux的搜索PATH中，因此终端无法找到该命令，此概念Windows通用 可能性：10% 解决方法，将可执行文件的目录通过export命令加入到PATH环境变量中：
> 通过修改.bashrc文件:
> vim ~/.bashrc 
> //在最后一行添上：
> export PATH=/usr/local/mongodb/bin:$PATH
> 生效方法：（有以下两种）
> 1、关闭当前终端窗口，重新打开一个新终端窗口就能生效
> 2、输入“source ~/.bashrc”命令，立即生效
> 有效期限：永久有效
> 用户局限：仅对当前用户。

> **更多问题和建议可以在下方评论区留言讨论，或者发我邮箱，邮箱在ABOUT中。原创帖转载请留言并注明出处**









