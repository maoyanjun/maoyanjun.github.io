---
layout:     post
title:      OpenFOAM-v1706中重叠网格的网格操作流程
subtitle:    ""
date:       2017-12-21
author:     Mao Yanjun
header-img: img/post-myj-oversetmesh.png
catalog: true
tags:
    - OpenFOAM
    - overset Mesh
---

> “🙉🙉🙉 ”

> **本帖会持续更新补充，新手所写，理论不严谨，代码不简洁，逻辑不清晰，仅供学习记录参考**

# 关于重叠网格的网格操作流程

以overInterDyMFoam 中的floatingobject Case为例简单介绍网格操作流程。因为overmesh的原理是使用两套网格，并通过overset区域进行插值传递信息。因此目录中有background 和 floatingObject 两个文件夹。其中，可以通过两者的Allrun.pre文件可以看到其在每个文件夹中的计算流程。简单说floatingObject只进行了浮体网格的设置和边界名称的设置。通过mergemesh到background网格中，剩下的操作都在background文件夹中进行。

Allrun 脚本实例和注释如下：

```
Allrun.pre IN floatingOject
#!/bin/sh
cd ${0%/*} || exit 1    # run from this directory

# Source tutorial run functions
. $WM_PROJECT_DIR/bin/tools/RunFunctions

# Set application name
application=`getApplication`

runApplication blockMesh #blockMesh定义了重叠区域
runApplication topoSet #topoSet定义了浮体的形状和位置的Cell 重叠域为除去浮体内部的所有Cell
runApplication subsetMesh -overwrite c0 -patch floatingObject #对c0域添加边界名字为floatingOject

# ----------------------------------------------------------------- end-of-file
Allrun.pre IN background
#!/bin/sh
cd ${0%/*} || exit 1    # Run from this directory
. $WM_PROJECT_DIR/bin/tools/RunFunctions

# Create background mesh
runApplication blockMesh

# Add the cylinder mesh
runApplication mergeMeshes . ../floatingBody –overwrite #关节的一步是将floatingBody的网格融合进了背景网格中。

# Select cellSets for the different zones
runApplication topoSet #下面详述：对计算域Cell进行了定义,便于更新域和setFields.

restore0Dir

# Use cellSets to write zoneID
runApplication setFields #设置初始场值，以及用（0，1）标记floatingbody的区域zoneID.

runApplication decomposePar
#------------------------------------------------------------------------------

```

# TopoSetDict 的使用和介绍

TopoSetDict:是在floatingBody中 由blockMesh定义的side region内进行设置，首先设置了c0为floatingobject边界内一点，则创建了新的cell zone 为c0.又以c0为基础创建了C1，再通过反选c0 zone的c1 cell zone.即floatingoject和side之间的区域。
![重叠网格示意图](https://i.imgur.com/iLUZM7I.png)

图中的overset Boundary 既是OF中定义的side,wall boundaries 则是floatingObject 边界。反选后的C1则是重叠区域

```
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      topoSetDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

actions
(
    {
        name    c0;
        type    cellSet;
        action  new;
        source  regionToCell;
        sourceInfo
        {
            insidePoints ((12.0 0.05 1.0));
        }
    }

    {
        name    c1;
        type    cellSet;
        action  new;
        source  cellToCell;
        sourceInfo
        {
            set c0;
        }
    }

    {
        name    c1;
        type    cellSet;
        action  invert;
    }
);

// ************************************************************************* //

```

# setField 对流体域进行定义

然后通过setField进行alphawater的设置和zoneID的设置。其中c0 zone 被定义为0，此区域不参加插值求解。被冻结？？。C1 zone 设置为1，即为参与插值求解的区域。此部分网格会随着sixdof进行网格的运动。同时通过此区域进行双向插值计算。（以上内容，纯属个人猜测总结，理论基础不深，后期会修正）

```
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      setFieldsDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

defaultFieldValues
(
    volScalarFieldValue alpha.water 0
    volScalarFieldValue zoneID 123
);

regions
(
    boxToCell
    {
        box ( -100 -100 -100 ) ( 100 100 1.0 );
        fieldValues ( volScalarFieldValue alpha.water 1 );
    }

 /*   boxToCell
    {
        box ( 0.7 0.8 -100 ) ( 100 100 0.75 );
        fieldValues ( volScalarFieldValue alpha.water 1 );
    }*/
    cellToCell
    {
        set c0;

        fieldValues
        (
            volScalarFieldValue zoneID 0
        );
    }
    cellToCell
    {
        set c1;

        fieldValues
        (
            volScalarFieldValue zoneID 1
        );
    }

);

// ************************************************************************* //

```
**最后关于topoSet, sefFieldsDict, refinemesh,等网格和前处理工具，可在代码中application/utilities/中看到详细的Dict设置指导。以及实现的源代码。**













