---
layout:     post
title:      OverInterDyMFoams使用心得
subtitle:    ""
date:       2018-1-4
author:     Mao Yanjun
header-img: img/post-myj-oversetmesh.png
catalog: true
tags:
    - OpenFOAM
    - overset Mesh
    - CFD
---

> “🙉🙉🙉 ”

> **本帖会持续更新补充，新手所写，理论不严谨，代码不简洁，逻辑不清晰，仅供学习记录参考**

# overInterDyMFoam使用心得
overInterDyMFoam是一个非常抗造的求解器，因为重叠网格的效果，所以一般只要边界条件和初始条件不出现严重问题的算例都可以进行下去，但是经过笔者进行的一些算例设置，重叠网格应用有一个较大的问题就是重叠区域网格设置有一些特殊的地方
* 重叠网格和背景网格差异不宜过大
* 重叠区域的结构物外边界要有足够厚度的网格进行与背景网格的插值和数据交换
* 容易出现局部插值不准的问题

# post process of the floatingObject with oversetMesh in  paraView 

Since the boundary patches do not update with the overSetMesh,so it is difficult to pick up the floatingObject from the internal meshes. As is known that the oversetMesh has to mark the hole, interpolation, and normal cells with 2 ,1,0. Hence, the Threshold can be used to show the hole cells which means the floatingObject. scalar by cell Types and limit the Minimum and Maximum value of the cellTypes both to 2. this will show the hole cells. further postprocess method will provide later.
 
# 测试内容 请忽略
* $a=xy+x^2$
* "行内公式"
* ```行内公式```
* ``行内公式``
```
多行共识
多行共识
```
`单引号行内测试`
`
单引号多行
单引号多行
`












