# foamNewAppから定常熱伝導ソルバーの実行まで
## 解く式
定常熱伝導方程式(ソース項なし)
$$
\nabla \cdot k\nabla T= 0
$$

## foamNewAppの実行
ターミナルで以下を実行し
```
mkdir -p $WM_PROJECT_USER_DIR/applications/myTests
cd $WM_PROJECT_USER_DIR/applications/myTests
foamNewApp myThermalConductionSolver
```
１行目、２行目でmyTestsフォルダを作成し、３行目で`foamNewApp`を用いてmyThermalConductionSolverを作成。この時点でファイル構成はこのようになる
```
myThermalConductionSolver
├── Make
│   ├── files
│   └── options
└── myThermalConductionSolver.C
```
## myThermalConductionSolver.Cの編集
includeの二行とsolve以下３行を追加
```
#include "fvCFD.H"

// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

int main(int argc, char *argv[])
{
    #include "setRootCase.H"
    #include "createTime.H"
    #include "createMesh.H"   //　この行を追加
    #include "createFields.H"  //　この行を追加

    // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

    solve( fvm::laplacian(k, T) );  //　この行を追加
    runTime++;                      //　この行を追加
    runTime.write();                //　この行を追加

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}
```
## createFields.Hの作成
myThermalConductionSolver直下にcreateFields.Hを作成する。中身は以下。
```
volScalarField T
(
    IOobject
    (
        "T",
        runTime.timeName(),
        mesh,
        IOobject::MUST_READ,
        IOobject::AUTO_WRITE
    ),
    mesh
);

IOdictionary transportProperties
(
    IOobject
    (
        "transportProperties",
        runTime.constant(),
        mesh,
        IOobject::MUST_READ_IF_MODIFIED,
        IOobject::NO_WRITE
    )
);

dimensionedScalar k
(
    "k",
    dimArea/dimTime,
    transportProperties
);
```
## wmakeによるコンパイル
```
cd myThermalConductionSolver
wmake
```
## caseファイルの作成
ターミナルで以下を実行
```
run
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity thermalSquare
cd thermalSquare
mv 0/U 0/T; rm 0/p
sed -i s/volVectorField/volScalarField/g 0/T
sed -i s/U/T/g 0/T
sed -i s/"1 -1 0"/"0 0 1"/g 0/T
sed -i s/"(0 0 0)"/0/g 0/T
sed -i s/"(1 0 0)"/1/g 0/T
sed -i s/"noSlip;"/"fixedValue; value uniform 0;"/g 0/T
sed -i s/icoFoam/myThermalConductionSolver/g system/controlDict
sed -i s/"0.005"/1/g system/controlDict
sed -i s/"20"/1/g system/controlDict
sed -i s/Euler/steadyState/g system/fvSchemesrc
sed -i s/U/T/g system/fvSolution
sed -i s/nu/k/g constant/transportProperties
sed -i s/"0.01"/"4e-05"/g constant/transportProperties
```
## caseの実行
```
blockMesh
myThermalConductionSolver
paraFoam
```
