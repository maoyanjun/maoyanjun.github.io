---
layout:     post
title:      interFoam 代码解析
subtitle:    ""
date:       2018-4-9
author:     Mao Yanjun
header-img: img/post-myj-waves2Foam.png
catalog: true
tags:
    - OpenFOAM
    - CFD
    - code
---

```
/*---------------------------------------------------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     |
    \\  /    A nd           | Copyright (C) 2011-2015 OpenFOAM Foundation
     \\/     M anipulation  |
-------------------------------------------------------------------------------
License
    This file is part of OpenFOAM.

    OpenFOAM is free software: you can redistribute it and/or modify it
    under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    OpenFOAM is distributed in the hope that it will be useful, but WITHOUT
    ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
    FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License
    for more details.

    You should have received a copy of the GNU General Public License
    along with OpenFOAM.  If not, see <http://www.gnu.org/licenses/>.

Application
    interFoam

Description
    Solver for 2 incompressible, isothermal immiscible fluids using a VOF
    (volume of fluid) phase-fraction based interface capturing approach.

    The momentum and other fluid properties are of the "mixture" and a single
    momentum equation is solved.

    Turbulence modelling is generic, i.e. laminar, RAS or LES may be selected.

    For a two-fluid approach see twoPhaseEulerFoam.

\*---------------------------------------------------------------------------*/

#include "fvCFD.H"    //有限体积法离散相关文件
#include "CMULES.H"  //采用MULES方法求解，OpenFOAM开发的一种半隐离散格式
#include "EulerDdtScheme.H"  //欧拉时间离散格式
#include "localEulerDdtScheme.H"
#include "CrankNicolsonDdtScheme.H"
#include "subCycle.H"  //调用子时间步
#include "immiscibleIncompressibleTwoPhaseMixture.H"  //不可压缩双相传输模型（alpha方程）
#include "turbulentTransportModel.H"  //湍流模型
#include "pimpleControl.H"
#include "fvIOoptionList.H"   //源项
#include "CorrectPhi.H"  //通量修正
#include "fixedFluxPressureFvPatchScalarField.H"  //设置边界压力条件
#include "localEulerDdtScheme.H"
#include "fvcSmooth.H"  //光顺求解

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"  //设置case目录
//    //
//// setRootCase.H
//// ~~~~~~~~~~~~~
//
//    Foam::argList args(argc, argv);
//    if (!args.checkRootCase())
//    {
//        Foam::FatalError.exit();
//    }

    #include "createTime.H"  //创建时间对象
//
// createTime.H
// ~~~~~~~~~~~~
//
//    Foam::Info<< "Create time\n" << Foam::endl;
//
//    Foam::Time runTime(Foam::Time::controlDictName, args);

    #include "createMesh.H"  //创建网格对象
    
//
// createMesh.H
// ~~~~~~~~~~~~
//
//    Foam::Info
//        << "Create mesh for time = "
//        << runTime.timeName() << Foam::nl << Foam::endl;
//
//    Foam::fvMesh mesh
//    (
//        Foam::IOobject
//        (
//            Foam::fvMesh::defaultRegion,
//            runTime.timeName(),
//            runTime,
//            Foam::IOobject::MUST_READ
//        )
//    );

///OpenFOAM/OpenFOAM-3.0.1/src/finiteVolume/cfdTools/general/solutionControl$
//pimpleControl  pisoControl  simpleControl  solutionControl

    pimpleControl pimple(mesh);  //采用pimple算法,参数：fvMesh& mesh, const word& dictName

    #include "createTimeControls.H"  //创建时间控制
    
//    
//    Global
//    readTimeControls
//
//Description
//    Read the control parameters used by setDeltaT
//
//\*---------------------------------------------------------------------------*/
//
//bool adjustTimeStep =
//    runTime.controlDict().lookupOrDefault("adjustTimeStep", false);
//
//scalar maxCo =
//    runTime.controlDict().lookupOrDefault<scalar>("maxCo", 1.0);
//
//scalar maxDeltaT =
//    runTime.controlDict().lookupOrDefault<scalar>("maxDeltaT", GREAT);

// ************************************************************************* //

    #include "createRDeltaT.H"  //创建时间步（LTS）
    #include "initContinuityErrs.H"  //初始连续性误差
    
//    Global
//    cumulativeContErr
//
//Description
//    Declare and initialise the cumulative continuity error.
//
//\*---------------------------------------------------------------------------*/
//
//#ifndef initContinuityErrs_H
//#define initContinuityErrs_H
//
//// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
//
//scalar cumulativeContErr = 0;
//
//// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
//
//#endif
//
//// ************************************************************************* //

    #include "createFields.H"  //创建场对象

    #include "createMRF.H"  //更新边界速度
    #include "createFvOptions.H"  //创建源项
    
    
//fv::IOoptionList fvOptions(mesh);

    
    #include "correctPhi.H"  //修正界面流率  保证连续性


    if (!LTS)  //如果是非局部时间步  （LTS的deltaT为非均一的场）,初始检验一次
    {
        #include "readTimeControls.H"  //读取时间控制
        #include "CourantNo.H"  //计算库朗数
        #include "setInitialDeltaT.H"  //初始时间步
    }

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< "\nStarting time loop\n" << endl;  //输出提示信息   开始时间循环

    while (runTime.run())
    {
        #include "readTimeControls.H"  //读取时间控制 第三次读取

        if (LTS)
        {
            #include "setRDeltaT.H"  //创建LTS的时间步
        }
        else  //进行每一run时间的检测后的时间步长的设置
        {
            #include "CourantNo.H"  //计算库朗数
            #include "alphaCourantNo.H"  //计算alpha库朗数
            #include "setDeltaT.H"  //设置时间步
        }

        runTime++;  //时间递增

        Info<< "Time = " << runTime.timeName() << nl << endl;  //输出当前运行时间点

        // --- Pressure-velocity PIMPLE corrector loop
        while (pimple.loop())    //进入速度压力pimple外循环
        {
            #include "alphaControls.H"  //读取alpha控制  （MULES方法控制、子循环次数、界面压缩因子等）
            #include "alphaEqnSubCycle.H"  //alpha方程求解

            mixture.correct();  //1、曲率修正  2、平均运动粘度修正
            
//  OpenFOAM-3.0.1/src/transportModels/immiscibleIncompressibleTwoPhaseMixture$         
//                 Member Functions
//
//        //- Correct the transport and interface properties
//        virtual void correct()
//        {
//            incompressibleTwoPhaseMixture::correct();

    //       //- Correct the laminar viscosity
    //        virtual void correct()
    //        {
    //            calcNu();
    //        }

        //// * * * * * * * * * * * * Private Member Functions  * * * * * * * * * * * * //
        //
        ////- Calculate and return the laminar viscosity
        //void Foam::incompressibleTwoPhaseMixture::calcNu()
        //{
        //    nuModel1_->correct();
        //    nuModel2_->correct();
        //
        //    const volScalarField limitedAlpha1
        //    (
        //        "limitedAlpha1",
        //        min(max(alpha1_, scalar(0)), scalar(1))
        //    );
        //
        //    // Average kinematic viscosity calculated from dynamic viscosity
        //    nu_ = mu()/(limitedAlpha1*rho1_ + (scalar(1) - limitedAlpha1)*rho2_);
        //}


//            interfaceProperties::correct();

        //             void Foam::interfaceProperties::calculateK()
        //{
        //    const fvMesh& mesh = alpha1_.mesh();
        //    const surfaceVectorField& Sf = mesh.Sf();//面法向矢量
        //
        //    // Cell gradient of alpha
        //    const volVectorField gradAlpha(fvc::grad(alpha1_, "nHat"));
        //
        //    // Interpolated face-gradient of alpha
        //    surfaceVectorField gradAlphaf(fvc::interpolate(gradAlpha));
        //
        //    //gradAlphaf -=
        //    //    (mesh.Sf()/mesh.magSf())
        //    //   *(fvc::snGrad(alpha1_) - (mesh.Sf() & gradAlphaf)/mesh.magSf());
        //
        //    // Face unit interface normal
        //    surfaceVectorField nHatfv(gradAlphaf/(mag(gradAlphaf) + deltaN_));
        //    // surfaceVectorField nHatfv
        //    // (
        //    //     (gradAlphaf + deltaN_*vector(0, 0, 1)
        //    //    *sign(gradAlphaf.component(vector::Z)))/(mag(gradAlphaf) + deltaN_)
        //    // );
        //    correctContactAngle(nHatfv.boundaryField(), gradAlphaf.boundaryField());//修正接触角？？
        //
        //    // Face unit interface normal flux
        //    nHatf_ = nHatfv & Sf;

        //    // Simple expression for curvature
        //    K_ = -fvc::div(nHatf_);
        //
        //    // Complex expression for curvature.
        //    // Correction is formally zero but numerically non-zero.
        //    /*
        //    volVectorField nHat(gradAlpha/(mag(gradAlpha) + deltaN_));
        //    forAll(nHat.boundaryField(), patchi)
        //    {
        //        nHat.boundaryField()[patchi] = nHatfv.boundaryField()[patchi];
        //    }
        //
        //    K_ = -fvc::div(nHatf_) + (nHat & fvc::grad(nHatfv) & nHat);
        //    */
        //}



//        }


            #include "UEqn.H"  //求解速度方程，若开启动量预测，则得到一个预测的U

            // --- Pressure corrector loop
            while (pimple.correct())  //内循环
            {
                #include "pEqn.H"  //求解压力方程，迭带计算一个时间步长内的P，p_rgh
            }

            if (pimple.turbCorr()) //如果控制湍流的开关不为零
            {
                turbulence->correct();  //修正湍动黏性
            }
        }

        runTime.write();  //写入数据

        Info<< "ExecutionTime = " << runTime.elapsedCpuTime() << " s"
            << "  ClockTime = " << runTime.elapsedClockTime() << " s"
            << nl << endl;
    }

    Info<< "End\n" << endl;

    return 0;
}


// ************************************************************************* //
```