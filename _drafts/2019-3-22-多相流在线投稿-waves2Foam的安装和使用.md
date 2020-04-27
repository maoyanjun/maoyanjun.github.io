---
layout:     post
title:      数值波浪水池构建工具waves2FOAM的安装与使用
subtitle:    ""
date:       2018-1-10
author:     Mao Yanjun 毛艳军
header-img: img/post-myj-waves2Foam.png
catalog: true
tags:
    - OpenFOAM
    - waves2Foam
    - CFD
    - NWT
---

# 背景介绍

在应用 CFD 方法进行 船舶与海洋工程，港口海岸及近海工程等相关工程问题模拟的过程中，首先要做的就是建立数值波浪水槽(Numerical Wave Tank NWT),NWT 需要具备基本的造波和消波功能。`waves2Foam` 就是一个基于 `OpenFOAM` 进行二次开发的用于波浪模拟的拓展工具箱。它是由丹麦科技大学的 Niels Gj$\phi$l Jacobsen 在2011年9月开源公布的，也是目前影响力和知名度较高的一个造波工具箱。`waves2Foam` 采用速度入口式造波方法，松弛区消波方式。预设了多种规则波，不规则波，孤立波等造波类型；松弛区消波可以在水槽两端设置先后消波区从而可以消除尾端波浪以及结构物二次反射波浪。经众多学者和以及笔者验证，`waves2Foam` 造波和消波效果都较为稳定，同时计算效率也表现不错，因此得到了众多学者使用。同时，其中包含了较多的前后处理程序也是值得学习和使用的。

!()[waves2foam_水槽示意图]

**通过本文你可以学习一下内容**

* waves2Foam 安装过程中容易出现的问题和解决办法，是waves2Foam手册的有效补充
* waveDyMFoam 和 overwaveDyMFoam 动网格版本求解器的手动修改，手册的有效补充和首发内容
* 新手学习和使用OF和waves2Foam的建议，很中肯，很重要

# waves2Foam的安装与使用

关于`wavesFoam`的安装与使用，[http://http://openfoamwiki.net/index.php/Contrib/waves2Foam](http://http://openfoamwiki.net/index.php/Contrib/waves2Foam "openfoamwiki"),上面已经做了详细的介绍，其中主要内容都迁移到了wave2Foam Manual 上去了。因此对于使用`waves2Foam`的人员，建议通读manualwaves2Foam.pdf。其中安装部分已经把基本安装流程和所需要的依赖都说明白了，先移步去尝试尝试可好？这里主要是点出安装过程中对于新手常遇到的几个小坑。

# 支持版本

手册中已经写明了支持的版本，亲测2.x ,3.x系列几乎安装没有太大问题，下面小坑有几个可以学习一下，万一不幸碰到也好解决，（为什么笔者这么不幸），~~目前阶段OF1706版本暂时编译不了~~。


## OpenFoam-v1706 版本支持修改

waves2Foam 近期公布了其支持的`OF17012_PLUS`版本，博主之前尝试了N久的在`OF1706_PLUS`上编译安装的工作也算告一段落，在此填坑，是否能够和重叠网格有效结合有待于进一步验证，应该问题不大。
由于暂时不会将`OF1706_PLUS`直接升级为`1712`，因此强迫症的我打算再次尝试`OF1706_PLUS`上编译安装。``application/solvers/``里面没有对应`1706_PLUS`的版本。因此从`1712_PLUS`版拷贝，从`1606`拷贝的兼容性极差，似乎版本差异太大，缺失很多声明文件。

此部分讲解较为详细，希望读者可以耐心阅读学习整个程序修改和查错过程，注意程序中的注释内容，逐渐具备基本的程序修改能力


```
cp -r solver1712_PLUS/ solver1706_PLUS/
```

设置好其他，编译发现如下错误：

```
1706/src/OSspecific/POSIX/lnInclude   -fPIC -c porousWaveFoam.C -o Make/linux64GccDPInt64Opt/porousWaveFoam.o
In file included from alphaEqnSubCycle.H:30:0,
                 from porousWaveFoam.C:116:
alphaEqn.H: In function ‘int main(int, char**)’:
alphaEqn.H:79:9: error: ‘scAlpha’ was not declared in this scope
     if (scAlpha > 0)
         ^
In file included from alphaEqnSubCycle.H:38:0,
                 from porousWaveFoam.C:116:
alphaEqn.H:79:9: error: ‘scAlpha’ was not declared in this scope
     if (scAlpha > 0)
         ^
In file included from porousWaveFoam.C:127:0:
UEqn.H:29:27: warning: unused variable ‘mu’ [-Wunused-variable]
     const volScalarField& mu = tmu();
                           ^
/home/shzx/OpenFOAM/OpenFOAM-v1706/wmake/rules/General/transform:28: recipe for target 'Make/linux64GccDPInt64Opt/porousWaveFoam.o' failed

```

显示为`scAlpha`未声明，查看此变量未找到相关声明，参考查找``icAlpha``关键字，在``OpenFOAM-v1706/src/finiteVolume/cfdTools/general/include/alphaControls$``中发现此变量，如下：

```
const dictionary& alphaControls = mesh.solverDict(alpha1.name());

label nAlphaCorr(readLabel(alphaControls.lookup("nAlphaCorr")));

label nAlphaSubCycles(readLabel(alphaControls.lookup("nAlphaSubCycles")));

bool MULESCorr(alphaControls.lookupOrDefault<Switch>("MULESCorr", false));

// Apply the compression correction from the previous iteration
// Improves efficiency for steady-simulations but can only be applied
// once the alpha field is reasonably steady, i.e. fully developed
bool alphaApplyPrevCorr
(
    alphaControls.lookupOrDefault<Switch>("alphaApplyPrevCorr", false)
);

// Isotropic compression coefficient
scalar icAlpha
(
    alphaControls.lookupOrDefault<scalar>("icAlpha", 0)
);

```

对比1712代码，发现了如下关键字声明：

```
// Isotropic compression coefficient
scalar icAlpha
(
    alphaControls.lookupOrDefault<scalar>("icAlpha", 0)
);

// Shear compression coefficient
scalar scAlpha
(
    alphaControls.lookupOrDefault<scalar>("scAlpha", 0)
);

```

因不方便改动源码，因此将此声明放到了alphaEqn.H中，尝试可以编译成功，至此此坑填满。撒花

如下：

```
// Shear compression coefficient
 scalar scAlpha
 (
     alphaControls.lookupOrDefault<scalar>("scAlpha", 0)
     );
    // Add the optional shear compression contribution
    if (scAlpha > 0)
    {
        phic +=
            scAlpha*mag(mesh.delta() & fvc::interpolate(symm(fvc::grad(U))));
    }

```



# 相关依赖

一定要把manualwaves2Foam中的相关依赖都下载和编译安装好之后再进行`waves2Foam`的安装。依赖的相关安装方法自行百度。这里面有个很容易出的问题，就是`ThirdParrty`里面，关于OceanWave3D-Fortran90的下载，与安装时采用的git clone.

``git clone https://github.com/boTerpPaulsen/OceanWave3D-Fortran90.git  ``,详细见``ThirdParty/Allwmake``

这里面关键部分是这部分容易出问题，如果`OceanWave3D`编译出现问题，请参考下面这段的介绍：

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

这一部分由于网络问题或者git 库可能迁移，会导致无法下载`OceanWave3D`,从而引发一系列问题。导致编译失败。此部分可以通过阅读`make` 文件的操作，进行手动解决`OceanWave3D`的编译，主要进行了以下操作：

* 下载`OceanWave3D` 到`ThirdParty`/目录下，可手动从github上搜索下载。
*  ``cp $settings/oceanWave3DSettings/common.mk $ocw/.``
*  ``cp $settings/oceanWave3DSettings/makefile $ocw/.``

完成这三步，就相当于手动解决了`OCeanWave3D`的下载问题，然后可以运行`Allmake`进行编译。或者自行阅读后续过程手动编译`OceanWave3D`.

# 环境变量设置

这里按照手册要求，安装路径是 ``$FOAM_RUN/../applications/utilities``，这里使用了OF的环境变量`FOAM_RUN`,新手一定要将源代码放入此路径中，因为默认的环境变量$`WAVES_DIR`是这个路径，当然，进一步如果了解到了这个问题，你可以把waves2Foam安装到任意位置。环境变量设置文件位于``\bin\bashrc.org``下面，此路径中定义了所有waves2Foam的环境变量。由于篇幅限制，完整文件请自行查看，下面展示主要部分： #号表示注释内容

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

**补充：使用默认的路径编译好可执行文件后，后续如果自己改动求解器编译时Make/file 中的路径尽量不要使用waves2Foam提供了的环境变量，最好使用绝对路径，虽然麻烦了一些但是不会出现太多的环境变量问题，或者如上文所述，记得将其添加到自己的配置文件中，每次开启终端即可自动source .bashrc的话也可以**

**4/20/2018 9:04:45 PM **

**经过再次编译安装前一定要到/bin/里面``source bashrc``,记得设置waves2Foam的环境变量，否则你会得到各种错误,如缺少很多include文件：以及如下问题：**

```
mkdir: cannot create directory ‘’: No such file or directory
/home/shzx/OpenFOAM/OpenFOAM-v1706/wmake/makefiles/general:140: recipe for target '/waveDyMFoam' failed
make: *** [/waveDyMFoam] Error 1

```

# 算例运行

默认算例可以通过运行`Allrun`命令来运行算例，此处可以通过阅读`Allrun`来了解`waves2Foam`算例运行过程中的执行的命令，可以手动按顺序执行。此处一个小坑，算例换到其他位置后无法运行`Allrun`。请看代码注释部分

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
...
...
...
runApplication postProcessWaves2Foam
```

# waveDyMFoam的修改

部分版本中`waveDyMFoam`需要自己通过`interDyMFoam`进行修改，此处的主要问题就是需要理解`Make`文件夹中的两个文件`file`和`option`

file:**.C文件文件名，和可执行文件的存放位置，注意此处最好手动写绝对路径。

```
waveFoam.C

EXE = $(WAVES_APPBIN)/waveFoam
```
`options`:源文件需要引用的相关代码和库文件，编译时，时常出现缺失某些文件的情况，主要是此处路径的问题，可以在此处添加，include进来，强调一点，一定要注意路径书写的正确性，不要换行书写，注意结尾的 \。
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
    -DOFVERSION=240 \ #对应版本号
    -DEXTBRANCH=0 \ #正常版本设为0 extend 版本设为1
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

**4/20/2018 9:09:58 PM 新增waveDyMFoam.C的改写方式**

由于waves2Foam中未对动网格求解器进行编写，此部分在manual中给出了部分解释，但是相对较难理解。manual中的compareFiles.sh后显示的内容如下，这里面主要是对比了所有的interFoam和waveFoam的区别，作为参考用于waveDyMFOAM的改写：

```
============ BEGIN: alphaCourantNo.H ============
diff: /home/shzx/OpenFOAM/OpenFOAM-v1706/applications/solvers/multiphase/interFoam/alphaCourantNo.H: No such file or
 directory

============= END: alphaCourantNo.H =============

============ BEGIN: alphaEqn.H ============
diff: /home/shzx/OpenFOAM/OpenFOAM-v1706/applications/solvers/multiphase/interFoam/alphaEqn.H: No such file or direc
tory

============= END: alphaEqn.H =============

============ BEGIN: alphaEqnSubCycle.H ============
diff: /home/shzx/OpenFOAM/OpenFOAM-v1706/applications/solvers/multiphase/interFoam/alphaEqnSubCycle.H: No such file
or directory

============= END: alphaEqnSubCycle.H =============

============ BEGIN: alphaSuSp.H ============

============= END: alphaSuSp.H =============

============ BEGIN: correctPhi.H ============

============= END: correctPhi.H =============

============ BEGIN: createAlphaFluxes.H ============                                                        [70/276]
diff: /home/shzx/OpenFOAM/OpenFOAM-v1706/applications/solvers/multiphase/interFoam/createAlphaFluxes.H: No such file
 or directory

============= END: createAlphaFluxes.H =============

============ BEGIN: createFields.H ============
84,86d83
< //volScalarField gh("gh", g & (mesh.C() - referencePoint));
< //surfaceScalarField ghf("ghf", g & (mesh.Cf() - referencePoint));
<
143,144d139
<
< relaxationZone relaxing(mesh, U, alpha1);

============= END: createFields.H =============

============ BEGIN: pEqn.H ============

============= END: pEqn.H =============

============ BEGIN: rhofs.H ============

============= END: rhofs.H =============

============ BEGIN: setDeltaT.H ============
diff: /home/shzx/OpenFOAM/OpenFOAM-v1706/applications/solvers/multiphase/interFoam/setDeltaT.H: No such file or dire
ctory

============= END: setDeltaT.H =============
============ BEGIN: setRDeltaT.H ============
diff: /home/shzx/OpenFOAM/OpenFOAM-v1706/applications/solvers/multiphase/interFoam/setRDeltaT.H: No such file or dir
ectory

============= END: setRDeltaT.H =============

============ BEGIN: UEqn.H ============

============= END: UEqn.H =============

============ BEGIN: waveFoam.C ============
56,58d55
< #include "relaxationZone.H"
< #include "externalWaveForcing.H"
<
62a60
>     #include "postProcess.H"
67d64
<
71,75d67
<
<     #include "readGravitationalAcceleration.H"
<     #include "readWaveProperties.H"
<     #include "createExternalWaveForcing.H"
<
80d71
<     #include "postProcess.H"
84,87c75,80
<     #include "readTimeControls.H"
<     #include "CourantNo.H"
<     #include "setInitialDeltaT.H"
<
---                                                                                                          [7/276]
>     if (!LTS)
>     {
>         #include "readTimeControls.H"
>         #include "CourantNo.H"
>         #include "setInitialDeltaT.H"
>     }
97,99c90,99
<         #include "CourantNo.H"
<         #include "alphaCourantNo.H"
<         #include "setDeltaT.H"
---
>         if (LTS)
>         {
>             #include "setRDeltaT.H"
>         }
>         else
>         {
>             #include "CourantNo.H"
>             #include "alphaCourantNo.H"
>             #include "setDeltaT.H"
>         }
105,106d104
<         externalWave->step();
<
113,114d110
<             relaxing.correct();
<
142,144d137
<
<     // Close down the external wave forcing in a nice manner
<     externalWave->close();
============= END: waveFoam.C =============


```

* 首先做最简单的工作,复制interDyMFoam/ 到waveDyMFoam/ 参见上文
* 修改file 和option 文件 参见上文
* 代码改写，给出了局部添加代码行号和上下文位置：

**6/4/2018 6:56:47 PM  修正waveDyMFoam.C 的改写代码**

waveDyMFoam.C

```
 52 #include "relaxationZone.H"
 53 #include "externalWaveForcing.H"

 67     #include "createDyMControls.H"
 68 
       //此部分关于调整creatFields.H与readWaveProperties.H 的方法是错误的，编译不会出错，但是在运行是，当注册 U 速度场时需要读取 ``waveProperties`` Dict, 此时readWaveProperties.H 还没有注册，
      //因此会导致没有 waveProperties Dict 的支持。因此需要在 U 注册之前，进行 #include "readWaveProperties.H",且 readWaveProperties.H 需要readGravitationalAcceleration.H 中声明 g.因此争取修改方法如修改后所示代码，将#include "readWaveProperties.H" 也放入到creatFields.H中。
 69       #include "createFields.H"  //~~此部分对代码顺序做了一定的调整，下面三个注释的文件放在了creatFields.H中，因为初始化P=p_rgh+rho*gh，需要调用，因此需要调整顺序，此部分大多数版本的改写中都会遇到此问题。manual中给出的解决方式没理解，似乎即使编译成功也会出现无法运行的问题，在此修改，可以编译成功，但是暂未进行算例验证。~~
 70 //    #include "readGravitationalAcceleration.H"
 71 //    #include "readhRef.H"
 72 //    #include "gh.H"
 73 //    #include "readWaveProperties.H"~~
 74     #include "createExternalWaveForcing.H"

124     ┆   Info<< "Time = " << runTime.timeName() << nl << endl;
125                 ┆
126                 ┆externalWave->step();
127                 ┆
128     ┆   // --- Pressure-velocity PIMPLE corrector loop

145     ┆   ┆   ┆   ┆   if (mesh.topoChanging())
146     ┆   ┆   ┆   ┆   {
147     ┆   ┆   ┆   ┆   ┆   talphaPhi1Corr0.clear(); //因为此solver/waveFoam/是从v-1712版本复制而来，导致createAlphaFluxes.H的下面定义，与从1706版本复制的interDyMFoam/中兼容，因此需要修改，原代码：talphaPhiCorr0.clear();
148     ┆   ┆   ┆   ┆   }

176     ┆   ┆   #include "alphaControls.H"
177     ┆   ┆   #include "alphaEqnSubCycle.H"
178
179             relaxing.correct();
180     ┆   ┆   mixture.correct();

202    // Close down the external wave forcing in a nice manner
203     externalWave->close();
204
205     Info<< "End\n" << endl;

```

creatFiels.H
```
  1 #include "createRDeltaT.H"
  2 #include "readGravitationalAcceleration.H"
  3 #include "readWaveProperties.H"
  4 #include "readhRef.H"
  5 #include "gh.H"
  6 Info<< "Reading field p_rgh\n" << endl;
	volScalarField p_rgh
	(
	    IOobject
	    (
	        "p_rgh",
	        runTime.timeName(),
	        mesh,
	        IOobject::MUST_READ,
	        IOobject::AUTO_WRITE
	    ),
	    mesh
	);


 84 //volScalarField gh("gh", g & (mesh.C() - referencePoint));//观察发现此部分似乎多余，在gh文件中似乎已经定义，因此comment out
 85 //surfaceScalarField ghf("ghf", g & (mesh.Cf() - referencePoint));//针对于参考压力和参考重力高度的内容有待于进一步的了解
 86
 87

```


createAlphaFluxes.H的定义如下：
```
 20 // MULES Correction
 21 tmp<surfaceScalarField> talphaPhi1Corr0;

```

总结，此次编译工作纯属自娱自乐，程序调整较大，~~个别地方有待于进一步验证~~，经过以上修改，验证后可以正确的将waves2Foam的造波条件应用于interDyMFoam 求解器。 对整个程序有了更进一步的认识，下一步工作将其应用于重叠网格模型overInterFoam和isoInterFoam的新型自由液面的压缩格式上。

# overWaveDyMFoam solver

相信此部分内容应该属于全网首发，其实没什么新鲜东西，一句话，搞清楚什么是造波边界条件，什么是动网格求解器，两者算法上应该是相互独立的，因此上面 waveDyMFoam solver 的修改流程完全适用于 overWaveDyMFoam solver, 此部分工作已经过验证可以正确应用，需要注意的是相关依赖和include 文件要搞清楚，作者建议 overWaveDyMFoam 求解器最好基于 overInterDyMFoam 进行修改，对于新手不要随意换目录，很多 include 文件搞不清楚很容易找不到。 编译前，记得去 ``source waves2Foam/bin/bashrc``,否则 好多造波的库是找不到的，另外此处 waves2Foam 的造波边界条件似乎和 OF -v1706中的 waveModel 边界条件名称有所冲突，因此可以将自带的 waveModel 库 comment out 掉。

**error 提示**

1.

```
 
	Dictionary entry for patch inlet not found
	file: /home/mk/OpenFOAM/mk-v1712/run/waveFlume/constant/waveProperties"
	#这个是去掉自带波浪库的问题
```

2.

```

	--> FOAM FATAL ERROR: 
	
	    request for dictionary waveProperties from objectRegistry region0 failed
	    available objects of type dictionary are
	
	7
	(
	MRFProperties
	turbulenceProperties
	fvSchemes
	fvSolution
	data
	dynamicMeshDict
	transportProperties
	)
	#这个是因为没把 读取波浪的头文件放到 creatFields 中的问题

```

如果你觉得本部分对你的研究有借鉴意义，可以使用如下引用：

	```
	@article{Maoyanjun2018,
	title = "CFD simulation of an integration system of Oscillating Buoy WEC with a fxed box-type breakwater",
	journal = "OpenFOAM Workshop 2018",
	author = "Yanjun Mao, Yong Cheng,  Gangjun Zhai",
	keywords = "Wave energy converter;Integrated system;OpenFOAM;Overset mesh;Nonlinear power take-off model"
	}
	```

# 新手建议，此处是重点

1. 阅读英文资料，网站，论坛，特别是针对于研究人员，这是使用OpenFOAM这样程序的基本要求 
2. 对于OpenFOAM的学习建议，建议Linux系统，软件一定要自己安装几遍，多去折腾一些，才会有更多的感悟和收获，此外，Linux系统建立在开源框架下，有更多优秀的开源程序是基于该系统下的，方便后期的程序使用。
3. 熟悉Linux系统的使用，尤其是环境变量的基本意义，就像是全局变量一样，对于软件安装和使用过程大有裨益，回头看来，这个概念其实在windows下也同样重要，只是被我们忽略了而已，在很多破解或者编译软件的过程过程中都会遇到这个概念的。
4. 戒骄戒躁
5. 对于新手，经常会遇到的``命令不存在``的错误提示，思考了一下，大概有一下几种可能参考：
    - 命令输入不对 可能性：30%  解决方法：核对命令，注意大小写
    - 软件安装失败，命令本身不存在 可能性：30% 解决方法：重新检查软件安装过程，是否出错
    - 部分命令因为版本或者特殊原因安装失败，此时可以根据自己的命令到存放可执行文件的文件夹，默认``$FOAM_USER_APPIN``和``$FOAM_USER_LIBBIN``中查看是否存在 可能性：20% 解决方法：如果不存在，则找到对应的源程序，重新尝试wmake进行编译，或者重装整个程序。
    - 编译的可执行文件没有加入到Linux的搜索PATH中，因此终端无法找到该命令，此概念Windows通用 可能性：10% 解决方法，将可执行文件的目录通过export命令加入到PATH环境变量中：
        > 通过修改.bashrc文件:
        > vim ~/.bashrc 
        > //在最后一行添上：
        > export PATH=/usr/local/mongodb/bin:$PATH
        > 生效方法：（有以下两种）
        > 1、关闭当前终端窗口，重新打开一个新终端窗口就能生效
        > 2、输入“source ~/.bashrc”命令，立即生效
        > 有效期限：永久有效
        > 用户局限：仅对当前用户。

相关资料推荐：

> https://openfoamwiki.net/index.php/Contrib/waves2Foam
> https://zhuanlan.zhihu.com/p/41758756
> https://zhuanlan.zhihu.com/p/37406110

> **更多问题和建议欢迎交流，本帖寻找英文版翻译志愿者，期待知识更大范围共享传播。 邮箱 maoyanjun_dut@foxmail.com。 原创帖，转载请联系平台管理人员**









