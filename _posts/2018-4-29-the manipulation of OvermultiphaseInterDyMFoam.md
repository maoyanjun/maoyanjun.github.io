---
layout:     post
title:      The Manipulation of OvermultiphaseInterDyMFoam
subtitle:    
date:       2018-5-4
author:     Mao Yanjun
header-img: img/post-myj-waves2Foam.png
catalog: true
tags:
    - Linux
    - C++
    - OpenFOAM
    - code
---
> Hello everyone, my blog will be written in English starting from this piece of post. Please forgive me for my poor English writing skills. There may be many grammar mistakes and unclear meanings. I will try my best to illustrate the steps logically and express my thought clearly. Further suggestions of anything about the post are welcomed, waiting for your comments in the bottom comment zone.

> the post is about the manipulation of the overmultiphaseInterDyMFoam solver. the idea of this manipulation is from a PhD student from Tianjin University named "木鱼-计算流体" in "OpenFOAM 开源计算千人群" and the author hope this work can give a reference to CFDers who want to use this solver for further research. 

## Introduction

Since the overInterDyMFoam has been presented in the OpenFOAM-V1706PLUS, it is feasible to finish this work. fist all, We do not need to get a full understanding of the code structure of the overInterDyMFoam.C or even more deeply understanding of the overset lib. Because the overInterDyMFoam is for the computation of the two phase flow with overset mesh method. the interaction between the first phase and the second phase is dealt with VOF method. Also the multiphaseInterFoam solver is also dealt with the same method. what's more, they have extremely similar code structure. So it is not very difficult to modify the code to multiphaseInterFoam solver.

## Modification the path  starts from multiphaseInterDyMFoam.C (fail)
There are two strategies to modify the code. First one, modification from the multiphaseInterDyMFoam. and add the overset lib to the solver. after trying, unfortunatly, I failure to do this modification. I got some errors
* 1. afer some comparision to the code ``overInterDyMFoam.C `` and ``multiphaseInterDyMFoam.C``. some including codes need to be add to the ``multiphaseInterDyMFoam.C`` .
the total code can be seen as below:

```
/*---------------------------------------------------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     |
    \\  /    A nd           | Copyright (C) 2011-2016 OpenFOAM Foundation
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
    multiphaseInterFoam

Group
    grpMultiphaseSolvers grpMovingMeshSolvers

Description
    Solver for n incompressible fluids which captures the interfaces and
    includes surface-tension and contact-angle effects for each phase, with
    optional mesh motion and mesh topology changes.

    Turbulence modelling is generic, i.e. laminar, RAS or LES may be selected.

\*---------------------------------------------------------------------------*/

#include "fvCFD.H"
#include "dynamicFvMesh.H"

#include "CMULES.H"
#include "EulerDdtScheme.H"
#include "localEulerDdtScheme.H"
#include "CrankNicolsonDdtScheme.H"
#include "subCycle.H"

#include "multiphaseMixture.H"
#include "turbulentTransportModel.H"
#include "pimpleControl.H"
#include "fvOptions.H"
#include "CorrectPhi.H"

#include "fvcSmooth.H"
#include "cellCellStencilObject.H"
#include "localMin.H"
#include "interpolationCellPoint.H"
#include "transform.H"
#include "fvMeshSubset.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "postProcess.H"

    #include "setRootCase.H"
    #include "createTime.H"
    #include "createDynamicFvMesh.H"
    #include "initContinuityErrs.H"
    #include "createControl.H"
    #include "createTimeControls.H"
    #include "createDyMControls.H"
    #include "createFields.H"
    #include "createFvOptions.H"

    volScalarField rAU
    (
        IOobject
        (
            "rAU",
            runTime.timeName(),
            mesh,
            IOobject::READ_IF_PRESENT,
            IOobject::AUTO_WRITE
        ),
        mesh,
        dimensionedScalar("rAUf", dimTime/rho.dimensions(), 1.0)
    );

    #include "correctPhi.H"
    #include "createUf.H"
    #include "CourantNo.H"
    #include "setInitialDeltaT.H"

    const surfaceScalarField& rhoPhi(mixture.rhoPhi());

    turbulence->validate();
    
////////////////////////////////////
    #include "setCellMask.H"
    #include "setInterpolatedCells.H"
/////////////////////////////////////////

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< "\nStarting time loop\n" << endl;

    while (runTime.run())
    {
        #include "readControls.H"
        #include "CourantNo.H"
        #include "alphaCourantNo.H"

        #include "setDeltaT.H"

        runTime++;

        Info<< "Time = " << runTime.timeName() << nl << endl;

        // --- Pressure-velocity PIMPLE corrector loop
        while (pimple.loop())
        {
            if (pimple.firstIter() || moveMeshOuterCorrectors)
            {
                scalar timeBeforeMeshUpdate = runTime.elapsedCpuTime();

                mesh.update();

                if (mesh.changing())
                {
                    Info<< "Execution time for mesh.update() = "
                        << runTime.elapsedCpuTime() - timeBeforeMeshUpdate
                        << " s" << endl;

                    gh = (g & mesh.C()) - ghRef;
                    ghf = (g & mesh.Cf()) - ghRef;
                    
////////////////////////////
               // Update cellMask field for blocking out hole cells
                 #include "setCellMask.H"
                #include "setInterpolatedCells.H"
////////////////////////////////////////
                }

                if (mesh.changing() && correctPhi)
                {
                    // Calculate absolute flux from the mapped surface velocity
                    phi = mesh.Sf() & Uf;

                    #include "correctPhi.H"

                    // Make the flux relative to the mesh motion
                    fvc::makeRelative(phi, U);

                    mixture.correct();
                }

                if (mesh.changing() && checkMeshCourantNo)
                {
                    #include "meshCourantNo.H"
                }
            }

            mixture.solve();
            rho = mixture.rho();

            #include "UEqn.H"

            // --- Pressure corrector loop
            while (pimple.correct())
            {
                #include "pEqn.H"
            }

            if (pimple.turbCorr())
            {
                turbulence->correct();
            }
        }

        runTime.write();

        Info<< "ExecutionTime = " << runTime.elapsedCpuTime() << " s"
            << "  ClockTime = " << runTime.elapsedClockTime() << " s"
            << nl << endl;
    }

    Info<< "End\n" << endl;

    return 0;
}


// ************************************************************************* //
```

after adding the `` #include "setCellMask.H", #include "setInterpolatedCells.H"`` to the corresponding location and correct the ``Make/option`` and `` Make/file``. we  can get the error as below: 

```
//error information
cellMask was not declared in this scope
cellMask.primitiveFieldRef() =1.0;
```

**Reason:** that is because the difference of the ``creatFields.H``
for the ``multiphaseInterDyMFoam.C``, it includes the ``creatFields.H`` from ``../creatFields.H`` details as below:

```
Info<< "Reading field p_rgh\n" << endl;
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

Info<< "Reading field U\n" << endl;
volVectorField U
(
    IOobject
    (
        "U",
        runTime.timeName(),
        mesh,
        IOobject::MUST_READ,
        IOobject::AUTO_WRITE
    ),
    mesh
);

#include "createPhi.H"

multiphaseMixture mixture(U, phi);

// Need to store rho for ddt(rho, U)
volScalarField rho
(
    IOobject
    (
        "rho",
        runTime.timeName(),
        mesh,
        IOobject::READ_IF_PRESENT
    ),
    mixture.rho()
);
rho.oldTime();

// Construct incompressible turbulence model
autoPtr<incompressible::turbulenceModel> turbulence
(
    incompressible::turbulenceModel::New(U, phi, mixture)
);


#include "readGravitationalAcceleration.H"
#include "readhRef.H"
#include "gh.H"


volScalarField p
(
    IOobject
    (
        "p",
        runTime.timeName(),
        mesh,
        IOobject::NO_READ,
        IOobject::AUTO_WRITE
    ),
    p_rgh + rho*gh
);

label pRefCell = 0;
scalar pRefValue = 0.0;
setRefCell
(
    p,
    p_rgh,
    pimple.dict(),
    pRefCell,
    pRefValue
);

if (p_rgh.needReference())
{
    p += dimensionedScalar
    (
        "p",
        p.dimensions(),
        pRefValue - getRefCellValue(p, pRefCell)
    );
}

mesh.setFluxRequired(p_rgh.name());

#include "createMRF.H"

```

It miss the declaration of the ``cellMask`` and the ``InterpolatedCells``  which is declared the ``../../interFoam/overInterDyMFoam/creatFields.H`` like the below code: copying ``../creatFields.H`` to the ``overmultiphaseInterDyMFoam/`` and adding the Overset specific part to the ``creatFields.H``. this problem will be fixed.


```


    //- Overset specific

    // Add solver-specific interpolations
    {
        dictionary oversetDict;
        oversetDict.add("U", true);
        oversetDict.add("p", true);
        oversetDict.add("HbyA", true);
        oversetDict.add("p_rgh", true);
        oversetDict.add("alpha1", true);
        oversetDict.add("minGradP", true);

        const_cast<dictionary&>
        (
            mesh.schemesDict()
        ).add
        (
            "oversetInterpolationRequired",
            oversetDict,
            true
        );
    }

    // Mask field for zeroing out contributions on hole cells
    #include "createCellMask.H"

    // Create bool field with interpolated cells
    #include "createInterpolatedCells.H"

```

But more serious problems occur as shown below, terrible confusion:  the main error type is ``undefined reference to Foam::cellCellStencilObject::typeName`` for missing some libs or disorder the libs. As for this question, It should because there are some difference in the ``UEqn.H`` and ``pEqn.H`` between both solver codes.  Further explanation, for the ``multiphaseInterDyMFoam.C``, it links the ``UEqn.H`` and ``pEqn.H`` from ``../../interFoam/``, but for ``../../interFoam/overInterDyMFoam/`` it has specifial ``UEqn.H`` and ``pEqn.H`` for the overset lib. The reader can check the difference of ``UEqn.H`` and ``pEqn.H``in between ``../../interFoam/`` and  ``../../interFoam/overInterDyMFoam/``. So maybe these differences are the troublemakers. you can fellow this path to finish the modification,cp the ``UEqn.H`` and ``pEqn.H`` from ``../../interFoam/overInterDyMFoam/`` to ``../overmultiphasInterDyMFoam/``. I think this problem maybe be solved or may have further errors. The author just stop here and chose another way which maybe sounds easy, actually more complicated. We just take a detour to the destination. Please keeping reading.

```
g++ -std=c++11 -m64 -DOPENFOAM_PLUS=1706 -Dlinux64 -DWM_ARCH_OPTION=64 -DWM_DP -DWM_LABEL_SIZE=64 -Wall -Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid-offsetof -O3  -DNoRepository -ftemplate-depth-100 -I . -I.. -I../../VoF -I../../interFoam/interDyMFoam -I../../interFoam -I../multiphaseMixture/lnInclude -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/transportModels -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/transportModels/incompressible/lnInclude -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/transportModels/interfaceProperties/lnInclude -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/TurbulenceModels/turbulenceModels/lnInclude -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/TurbulenceModels/incompressible/lnInclude -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/dynamicMesh/lnInclude -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/dynamicFvMesh/lnInclude -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/finiteVolume/lnInclude -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/overset/lnInclude -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/meshTools/lnInclude -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/sampling/lnInclude -IlnInclude -I. -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/OpenFOAM/lnInclude -I/home/shzx/OpenFOAM/OpenFOAM-v1706/src/OSspecific/POSIX/lnInclude   -fPIC -Xlinker --add-needed -Xlinker --no-as-needed /home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o -L/home/shzx/OpenFOAM/OpenFOAM-v1706/platforms/linux64GccDPInt64Opt/lib \
    -lmultiphaseInterFoam -linterfaceProperties -lincompressibleTransportModels -lturbulenceModels -lincompressibleTurbulenceModels -lfiniteVolume -ldynamicMesh -ldynamicFvMesh -ltopoChangerFvMesh -lfvOptions -lmeshTools -lsampling -lOpenFOAM -ldl  \
     -lm -o /home/shzx/OpenFOAM/shzx-v1706/platforms/linux64GccDPInt64Opt/bin/overmultiphaseInterDyMFoam
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject::type() const':
OvermultiphaseInterDyMFoam.C:(.text._ZNK4Foam21cellCellStencilObject4typeEv[_ZNK4Foam21cellCellStencilObject4typeEv]+0x3): undefined reference to `Foam::cellCellStencilObject::typeName'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject::update()':
OvermultiphaseInterDyMFoam.C:(.text._ZN4Foam21cellCellStencilObject6updateEv[_ZN4Foam21cellCellStencilObject6updateEv]+0x23): undefined reference to `typeinfo for Foam::cellCellStencil'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject::cellInterpolationWeight() const':
OvermultiphaseInterDyMFoam.C:(.text._ZNK4Foam21cellCellStencilObject23cellInterpolationWeightEv[_ZNK4Foam21cellCellStencilObject23cellInterpolationWeightEv]+0x23): undefined reference to `typeinfo for Foam::cellCellStencil'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject::~cellCellStencilObject()':
OvermultiphaseInterDyMFoam.C:(.text._ZN4Foam21cellCellStencilObjectD2Ev[_ZN4Foam21cellCellStencilObjectD5Ev]+0x39): undefined reference to `Foam::cellCellStencil::~cellCellStencil()'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject::movePoints()':
OvermultiphaseInterDyMFoam.C:(.text._ZN4Foam21cellCellStencilObject10movePointsEv[_ZN4Foam21cellCellStencilObject10movePointsEv]+0x23): undefined reference to `typeinfo for Foam::cellCellStencil'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject::~cellCellStencilObject()':
OvermultiphaseInterDyMFoam.C:(.text._ZN4Foam21cellCellStencilObjectD0Ev[_ZN4Foam21cellCellStencilObjectD5Ev]+0x39): undefined reference to `Foam::cellCellStencil::~cellCellStencil()'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject::cellTypes() const':
OvermultiphaseInterDyMFoam.C:(.text._ZNK4Foam21cellCellStencilObject9cellTypesEv[_ZNK4Foam21cellCellStencilObject9cellTypesEv]+0x23): undefined reference to `typeinfo for Foam::cellCellStencil'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject::interpolationCells() const':
OvermultiphaseInterDyMFoam.C:(.text._ZNK4Foam21cellCellStencilObject18interpolationCellsEv[_ZNK4Foam21cellCellStencilObject18interpolationCellsEv]+0x23): undefined reference to `typeinfo for Foam::cellCellStencil'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject::cellInterpolationMap() const':
OvermultiphaseInterDyMFoam.C:(.text._ZNK4Foam21cellCellStencilObject20cellInterpolationMapEv[_ZNK4Foam21cellCellStencilObject20cellInterpolationMapEv]+0x23): undefined reference to `typeinfo for Foam::cellCellStencil'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject::cellStencil() const':
OvermultiphaseInterDyMFoam.C:(.text._ZNK4Foam21cellCellStencilObject11cellStencilEv[_ZNK4Foam21cellCellStencilObject11cellStencilEv]+0x23): undefined reference to `typeinfo for Foam::cellCellStencil'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject::cellInterpolationWeights() const':
OvermultiphaseInterDyMFoam.C:(.text._ZNK4Foam21cellCellStencilObject24cellInterpolationWeightsEv[_ZNK4Foam21cellCellStencilObject24cellInterpolationWeightsEv]+0x23): undefined reference to `typeinfo for Foam::cellCellStencil'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::cellCellStencilObject const& Foam::objectRegistry::lookupObject<Foam::cellCellStencilObject>(Foam::word const&, bool) const':
OvermultiphaseInterDyMFoam.C:(.text._ZNK4Foam14objectRegistry12lookupObjectINS_21cellCellStencilObjectEEERKT_RKNS_4wordEb[_ZNK4Foam14objectRegistry12lookupObjectINS_21cellCellStencilObjectEEERKT_RKNS_4wordEb]+0x99): undefined reference to `Foam::cellCellStencilObject::typeName'
OvermultiphaseInterDyMFoam.C:(.text._ZNK4Foam14objectRegistry12lookupObjectINS_21cellCellStencilObjectEEERKT_RKNS_4wordEb[_ZNK4Foam14objectRegistry12lookupObjectINS_21cellCellStencilObjectEEERKT_RKNS_4wordEb]+0x265): undefined reference to `Foam::cellCellStencilObject::typeName'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o: In function `Foam::MeshObject<Foam::fvMesh, Foam::MoveableMeshObject, Foam::cellCellStencilObject>::New(Foam::fvMesh const&)':
OvermultiphaseInterDyMFoam.C:(.text._ZN4Foam10MeshObjectINS_6fvMeshENS_18MoveableMeshObjectENS_21cellCellStencilObjectEE3NewERKS1_[_ZN4Foam10MeshObjectINS_6fvMeshENS_18MoveableMeshObjectENS_21cellCellStencilObjectEE3NewERKS1_]+0x3a): undefined reference to `Foam::cellCellStencilObject::typeName'
OvermultiphaseInterDyMFoam.C:(.text._ZN4Foam10MeshObjectINS_6fvMeshENS_18MoveableMeshObjectENS_21cellCellStencilObjectEE3NewERKS1_[_ZN4Foam10MeshObjectINS_6fvMeshENS_18MoveableMeshObjectENS_21cellCellStencilObjectEE3NewERKS1_]+0xf3): undefined reference to `Foam::cellCellStencil::cellCellStencil(Foam::fvMesh const&)'
OvermultiphaseInterDyMFoam.C:(.text._ZN4Foam10MeshObjectINS_6fvMeshENS_18MoveableMeshObjectENS_21cellCellStencilObjectEE3NewERKS1_[_ZN4Foam10MeshObjectINS_6fvMeshENS_18MoveableMeshObjectENS_21cellCellStencilObjectEE3NewERKS1_]+0x14a): undefined reference to `Foam::cellCellStencil::New(Foam::fvMesh const&, Foam::dictionary const&)'
OvermultiphaseInterDyMFoam.C:(.text._ZN4Foam10MeshObjectINS_6fvMeshENS_18MoveableMeshObjectENS_21cellCellStencilObjectEE3NewERKS1_[_ZN4Foam10MeshObjectINS_6fvMeshENS_18MoveableMeshObjectENS_21cellCellStencilObjectEE3NewERKS1_]+0x227): undefined reference to `Foam::cellCellStencil::~cellCellStencil()'
/home/shzx/OpenFOAM/OpenFOAM-v1706/build/linux64GccDPInt64Opt/applications/solvers/multiphase/multiphaseInterFoam/OvermultiphaseInterDyMFoam/OvermultiphaseInterDyMFoam.o:(.data.rel.ro._ZTIN4Foam21cellCellStencilObjectE[_ZTIN4Foam21cellCellStencilObjectE]+0x28): undefined reference to `typeinfo for Foam::cellCellStencil'
collect2: error: ld returned 1 exit status
/home/shzx/OpenFOAM/OpenFOAM-v1706/wmake/makefiles/general:140: recipe for target '/home/shzx/OpenFOAM/shzx-v1706/platforms/linux64GccDPInt64Opt/bin/overmultiphaseInterDyMFoam' failed
make: *** [/home/shzx/OpenFOAM/shzx-v1706/platforms/linux64GccDPInt64Opt/bin/overmultiphaseInterDyMFoam] Error 1
```
## The path starts from the overInterDyMFoam (success)

this path starts from the ``overInterDyMFoam``, and turn it into the ``overMultiphaseInterDyMFoam``.it seems more difficult to do there work. but I success to compile it.the multiphase model is more easy to change under the frame of interDyMFoam.C
What I have done are listed below:
1. modify the option and files in Make/

```
cp -r overInterDyMFoam/ overMultiphaseInterDyMFoam/
cd overMultiphaseInterDyMFoam/
mv overInterDyMFoam.C OvermultiphaseInterDyMFoam.C

```

### Modification of options and files
Compare the Make/option from multiphaseInterDyMFoam/ and overInterDyMFoam/, delete the linking files and compiled libs for twophase model, add the link file and compiled libs necessary for multiphase model. Both files are shown as below. the readers can compare them and check what modifications have been done.

options from overInterDyMFoam/

```

EXE_INC = \
    -I. \
    -I.. \
    -I../../VoF \
    -I$(LIB_SRC)/transportModels/twoPhaseMixture/lnInclude \
    -I$(LIB_SRC)/transportModels \
    -I$(LIB_SRC)/transportModels/incompressible/lnInclude \
    -I$(LIB_SRC)/transportModels/interfaceProperties/lnInclude \
    -I$(LIB_SRC)/TurbulenceModels/turbulenceModels/lnInclude \
    -I$(LIB_SRC)/TurbulenceModels/incompressible/lnInclude \
    -I$(LIB_SRC)/transportModels/immiscibleIncompressibleTwoPhaseMixture/lnInclude \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/dynamicMesh/lnInclude \
    -I$(LIB_SRC)/dynamicFvMesh/lnInclude \
    -I$(FOAM_SOLVERS)/incompressible/pimpleFoam/overPimpleDyMFoam \
    -I$(LIB_SRC)/overset/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude \
    -I$(LIB_SRC)/sampling/lnInclude

EXE_LIBS = \
    -limmiscibleIncompressibleTwoPhaseMixture \
    -lturbulenceModels \
    -lincompressibleTurbulenceModels \
    -lfiniteVolume \
    -ldynamicMesh \
    -ldynamicFvMesh \
    -ltopoChangerFvMesh \
    -loverset \
    -lfvOptions \
    -lsampling \
    -lwaveModels

``` 

options from overmultiphaseInterDyMFoam/

```

EXE_INC = \
    -I. \
    -I.. \
    -I../../VoF \
    -I../multiphaseMixture\
    -I$(LIB_SRC)/transportModels \
    -I$(LIB_SRC)/transportModels/incompressible/lnInclude \
    -I$(LIB_SRC)/transportModels/interfaceProperties/lnInclude \
    -I$(LIB_SRC)/TurbulenceModels/turbulenceModels/lnInclude \
    -I$(LIB_SRC)/TurbulenceModels/incompressible/lnInclude \
    -I$(LIB_SRC)/finiteVolume/lnInclude \
    -I$(LIB_SRC)/dynamicMesh/lnInclude \
    -I$(LIB_SRC)/dynamicFvMesh/lnInclude \
    -I$(FOAM_SOLVERS)/incompressible/pimpleFoam/overPimpleDyMFoam \
    -I$(LIB_SRC)/overset/lnInclude \
    -I$(LIB_SRC)/meshTools/lnInclude \
    -I$(LIB_SRC)/sampling/lnInclude

EXE_LIBS = \
    -lmultiphaseInterFoam \
    -linterfaceProperties \
    -lincompressibleTransportModels \
    -lturbulenceModels \
    -lincompressibleTurbulenceModels \
    -lfiniteVolume \
    -ldynamicMesh \
    -ldynamicFvMesh \
    -ltopoChangerFvMesh \
    -loverset \
    -lfvOptions \
    -lsampling \
    -lwaveModels

``` 


files from  overInterDyMFoam/

```
interDyMFoam.C

EXE = $(FOAM_APPBIN)/overInterDyMFoam
```


files from  overmultiphaseInterDyMFoam/

```
OvermultiphaseInterDyMFoam.C

EXE = $(FOAM_USER_APPBIN)/OvermultiphaseInterDyMFoam
```

### Modifications of  OvermultiphaseInterDyMFoam.C

```
/*---------------------------------------------------------------------------*\
  =========                 |
  \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
   \\    /   O peration     |
    \\  /    A nd           | Copyright (C) 2011-2016 OpenFOAM Foundation
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
    multiphaseInterFoam

Group
    grpMultiphaseSolvers grpMovingMeshSolvers

Description
    Solver for n incompressible fluids which captures the interfaces and
    includes surface-tension and contact-angle effects for each phase, with
    optional mesh motion and mesh topology changes.

    Turbulence modelling is generic, i.e. laminar, RAS or LES may be selected.

\*---------------------------------------------------------------------------*/

#include "fvCFD.H"
#include "dynamicFvMesh.H"

#include "CMULES.H"
#include "EulerDdtScheme.H"
#include "localEulerDdtScheme.H"
#include "CrankNicolsonDdtScheme.H"
#include "subCycle.H"

#include "multiphaseMixture.H"
#include "turbulentTransportModel.H"
#include "pimpleControl.H"
#include "fvOptions.H"
#include "CorrectPhi.H"

#include "fvcSmooth.H"
#include "cellCellStencilObject.H"
#include "localMin.H"
#include "interpolationCellPoint.H"
#include "transform.H"
#include "fvMeshSubset.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "postProcess.H"

    #include "setRootCase.H"
    #include "createTime.H"
    #include "createDynamicFvMesh.H"
    #include "initContinuityErrs.H"
    #include "createControl.H"
    #include "createTimeControls.H"
    #include "createDyMControls.H"
    #include "createFields.H"
    #include "createFvOptions.H"

    volScalarField rAU
    (
        IOobject
        (
            "rAU",
            runTime.timeName(),
            mesh,
            IOobject::READ_IF_PRESENT,
            IOobject::AUTO_WRITE
        ),
        mesh,
        dimensionedScalar("rAUf", dimTime/rho.dimensions(), 1.0)
    );

    #include "correctPhi.H"
    #include "createUf.H"
    #include "CourantNo.H"
    #include "setInitialDeltaT.H"

    const surfaceScalarField& rhoPhi(mixture.rhoPhi());

    turbulence->validate();
    
    ////////////////////////////////////
    #include "setCellMask.H"
    #include "setInterpolatedCells.H"
/////////////////////////////////////////

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    Info<< "\nStarting time loop\n" << endl;

    while (runTime.run())
    {
        #include "readControls.H"
        #include "CourantNo.H"
        #include "alphaCourantNo.H"

        #include "setDeltaT.H"

        runTime++;

        Info<< "Time = " << runTime.timeName() << nl << endl;

        // --- Pressure-velocity PIMPLE corrector loop
        while (pimple.loop())
        {
            if (pimple.firstIter() || moveMeshOuterCorrectors)
            {
                scalar timeBeforeMeshUpdate = runTime.elapsedCpuTime();

                mesh.update();

                if (mesh.changing())
                {
                    Info<< "Execution time for mesh.update() = "
                        << runTime.elapsedCpuTime() - timeBeforeMeshUpdate
                        << " s" << endl;

                    gh = (g & mesh.C()) - ghRef;
                    ghf = (g & mesh.Cf()) - ghRef;
                    
////////////////////////////
               // Update cellMask field for blocking out hole cells
                 #include "setCellMask.H"
                #include "setInterpolatedCells.H"
////////////////////////////////////////
                }

                if (mesh.changing() && correctPhi)
                {
                    // Calculate absolute flux from the mapped surface velocity
                    phi = mesh.Sf() & Uf;

                    #include "correctPhi.H"

                    // Make the flux relative to the mesh motion
                    fvc::makeRelative(phi, U);

                    mixture.correct();
                }

                if (mesh.changing() && checkMeshCourantNo)
                {
                    #include "meshCourantNo.H"
                }
            }

            mixture.solve();
            rho = mixture.rho();

            #include "UEqn.H"

            // --- Pressure corrector loop
            while (pimple.correct())
            {
                #include "pEqn.H"
            }

            if (pimple.turbCorr())
            {
                turbulence->correct();
            }
        }

        runTime.write();

        Info<< "ExecutionTime = " << runTime.elapsedCpuTime() << " s"
            << "  ClockTime = " << runTime.elapsedClockTime() << " s"
            << nl << endl;
    }

    Info<< "End\n" << endl;

    return 0;
}


// ************************************************************************* //
```

### Modifications of  creatFields.H

this creatFields.H is for the twophase model. so I combined the creatFields.Hs of both  in local dictionary and ``../multiphaseInterFoam/``

```
Info<< "Reading field p_rgh\n" << endl;
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

Info<< "Reading field U\n" << endl;
volVectorField U
(
    IOobject
    (
        "U",
        runTime.timeName(),
        mesh,
        IOobject::MUST_READ,
        IOobject::AUTO_WRITE
    ),
    mesh
);

#include "createPhi.H"


    //- Overset specific

    // Add solver-specific interpolations
    {
        dictionary oversetDict;
        oversetDict.add("U", true);
        oversetDict.add("p", true);
        oversetDict.add("HbyA", true);
        oversetDict.add("p_rgh", true);
        oversetDict.add("alpha1", true);
        oversetDict.add("minGradP", true);

        const_cast<dictionary&>
        (
            mesh.schemesDict()
        ).add
        (
            "oversetInterpolationRequired",
            oversetDict,
            true
        );
    }

    // Mask field for zeroing out contributions on hole cells
    #include "createCellMask.H"

    // Create bool field with interpolated cells
    #include "createInterpolatedCells.H"



multiphaseMixture mixture(U, phi);

// Need to store rho for ddt(rho, U)
volScalarField rho
(
    IOobject
    (
        "rho",
        runTime.timeName(),
        mesh,
        IOobject::READ_IF_PRESENT
    ),
    mixture.rho()
);
rho.oldTime();

// Construct incompressible turbulence model
autoPtr<incompressible::turbulenceModel> turbulence
(
    incompressible::turbulenceModel::New(U, phi, mixture)
);


#include "readGravitationalAcceleration.H"
#include "readhRef.H"
#include "gh.H"


volScalarField p
(
    IOobject
    (
        "p",
        runTime.timeName(),
        mesh,
        IOobject::NO_READ,
        IOobject::AUTO_WRITE
    ),
    p_rgh + rho*gh
);

label pRefCell = 0;
scalar pRefValue = 0.0;
setRefCell
(
    p,
    p_rgh,
    pimple.dict(),
    pRefCell,
    pRefValue
);

if (p_rgh.needReference())
{
    p += dimensionedScalar
    (
        "p",
        p.dimensions(),
        pRefValue - getRefCellValue(p, pRefCell)
    );
}

mesh.setFluxRequired(p_rgh.name());

#include "createMRF.H"

```

### cp multiphaseMixture to the ../interFoam/

Finally for easy searching the multiphaseMixture model, I just copy it from the ``../multiphaseInterFoam/`` to ``../interFoam/``

```
cp -r ../multiphaseInterFoam/multiphaseMixture ./
```

then, it will be successful to compile the OvermultiphaseInterDyMFoam.

```
wmake
```

### success 

After that, I am sure that I take a detour to finish this work. maybe in the first part, we almost get to the destination. but I do not have time to take a try. if you are working on it. just try it like  what I do in the first part.

**there is two things needed to declare:**

* Though It was compiled successfully, I have not done verfication of this solver. I do not know whether it works well or not. Hope feedbacks about it from the readers. 
* if you think this post help you with your research work, hope you can reference it as fellow format, if you want the total Codes, you can contact me with the Email. I will also upload  the codes to my github repository.[https://github.com/maoyanjun](https://github.com/maoyanjun)


```
@misc{OvermultiphaseInterDyMFoam-Yanjun Mao,
  author = {Mao Yanjun},
  title = {{The Manipulation of OvermultiphaseInterDyMFoam}},
  howpublished = "\url{http://maoyanjun.top/2018/05/04/The Manipulation of OvermultiphaseInterDyMFoam/}",
  year = {2018}, 
  note = "[Online; accessed 04-May-2018]"
}
```

> **Furether questions or suggestions can be leaved on the below comment zone or connect me by the Email. The Email address is in the ABOUT zone. the post is originally writen by Mao yanjun, If you want to reprint, please mark the reference.**










