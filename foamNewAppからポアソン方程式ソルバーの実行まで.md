# foamNewAppからポアソン方程式ソルバーの実行まで
## 解く式
ポアソン方程式方程式  
$\Delta \phi= -\rho/\epsilon$

## foamNewAppの実行
ターミナルで以下を実行し
```
mkdir -p $WM_PROJECT_USER_DIR/applications/myTests
cd $WM_PROJECT_USER_DIR/applications/myTests
foamNewApp myPoisonEqSolver
```
１行目、２行目でmyTestsフォルダを作成し、３行目で`foamNewApp`を用いてmyPoisonEqSolverを作成。この時点でファイル構成はこのようになる
```
myPoisonEqSolver
├── Make
│   ├── files
│   └── options
└── myPoisonEqSolver.C
```
## myPoisonEqSolver.Cの編集
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

    solve( fvm::laplacian(phi) + rho/epsiron );  //　この行を追加
    runTime++;                      //　この行を追加
    runTime.write();                //　この行を追加

    Info<< nl;
    runTime.printExecutionTime(Info);

    Info<< "End\n" << endl;

    return 0;
}
```
## createFields.Hの作成
myPoisonEqSolver直下にcreateFields.Hを作成する。中身は以下。
電解密度の場を与えてあげるために、２つ目のブロックでvolScalarField rhoを定義する。IOobjectの２番めの引数で定数であることを指示、４番目で読み込みの規則を、５番目で書き込みの規則を指示。

今回はtransportPorpertiesではなくfieldPropertiesというファイルを作って、そこからepsilonを読み込むこととする。epsilonの定義の記述がtransportPropertiesのmuと異なるので注意
```
volScalarField phi
(
    IOobject
    (
        "phi",
        runTime.timeName(),
        mesh,
        IOobject::MUST_READ,
        IOobject::AUTO_WRITE
    ),
    mesh
);

volScalarField rho
(
    IOobject
    (
        "rho",
        runTime.constant(),
        mesh,
        IOobject::MUST_READ_IF_MODIFIED,
        IOobject::AUTO_WRITE
    ),
    mesh
);

IOdictionary fieldProperties
(
    IOobject
    (
        "fieldProperties",
        runTime.constant(),
        mesh,
        IOobject::MUST_READ,
        IOobject::NO_WRITE
    )
);

dimensionedScalar epsilon
(
    fieldProperties.lookup("epsilon")
);
```
## wmakeによるコンパイル
```
cd myPoisonEqSolver
wmake
```
## caseファイルの作成
cavityチュートリアルをもとに作成する。\$FOAM_RUN ディレクトリ配下にsimPoisonEqというファイル名でコピー。ファイル名を変更する。変数の設定ファイルも作成。

ターミナルで以下を実行
```
run
cp -r $FOAM_TUTORIALS/incompressible/icoFoam/cavity/cavity simPoisonEq
cd simPoisonEq
mv 0/p 0/phi;
rm 0/U;
cp constant/transportProperties constant/rho
mv constant/transportProperties constant/fieldProperties
touch system/setFieldsDict
```
実行後のファイル構成は以下。epsilonとrhoは定数なのでconstant配下に保存。rhoをsetFieldsという前処理で定義するためのsetFieldsDictをsystemに配置。
```
simPoisonEq
├── 0
│   └── phi
├── constant
│   ├── fieldProperties
│   └── rho
└── system
    ├── blockMeshDict
    ├── controlDict
    ├── fvSchemes
    ├── fvSolution
    └── setFieldsDict
```
## phiの設定
FoamFileの変数名、次元、境界条件を変更。
```
FoamFile
{
    version     2.0;
    format      ascii;
    class       volScalarField;
    object      phi;　　　　　　　　　　　 // p => phi
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

dimensions      [1 2 -3 0 0 -1 0];     // 次元を変更

internalField   uniform 0;

boundaryField
{
    movingWall
    {
        type            fixedValue;    // zeroGradient => fixedValue
        value           uniform 0;     // zeroGradient => fixedValue
    }

    fixedWalls
    {
        type            fixedValue;    // zeroGradient => fixedValue
        value           uniform 0;     // zeroGradient => fixedValue
    }

    frontAndBack
    {
        type            empty;
    }
}
```
## rhoの変更
FoamFileのクラス名と変数名を変更。次元の追加と変数の値を記述。rhoの値はsetFieldsという前処理で定義するため、一旦0としておく。
```
FoamFile
{
    version     2.0;
    format      ascii;
    class       volScalarField;      // dictionary => volScalarField
    location    "constant";
    object      rho;　　　　　　　　　　// transportProperties => rho
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

dimensions      [0 -3 1 0 0 1 0];    // 次元を追加

rho             0;                   // setFieldsで定義するので0としておく

```

## fieldProperties(epsilon)の変更
FoamFileのオブジェクト名を変更。変数の次元と値を記述。tranportPropertiesではmuの次元は別の箇所で指示しているようだったが、epslonはここで記述した。
```
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "constant";
    object      fieldProperties;    　　　　　// object名を変更
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

epsiron            epsiron [-1 -3 4 0 0 2 0] 8.85e-12;  // 次元と値を記述

```

## blockMeshDictの変更
blockMeshDictのscaleの値を0.1から1に、メッシュ数を100×100に変更。計算領域を広げただけ。
```
scale   1;    // 0.1 => 1

vertices
(
    (0 0 0)
    (1 0 0)
    (1 1 0)
    (0 1 0)
    (0 0 0.1)
    (1 0 0.1)
    (1 1 0.1)
    (0 1 0.1)
);

blocks
(
    hex (0 1 2 3 4 5 6 7) (100 100 1) simpleGrading (1 1 1)  ]
    //(20 20 1) => (100 100 1)
);
```

## setFieldsDictの編集とsetFieldsの実行
rhoの値を指定するために前処理setFieldsを使用する。その設定ファイルとしてsetFieldsDictを記述しておく。cylinderToCellによって円柱を指定してその内部の値を設定するもの。p1とp2で指定された中心線から半径radiusを取る円柱を指定している。
```
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "system";
    object      setFieldsDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

defaultFieldValues
(
    volScalarFieldValue rho 0
);

regions
(
    cylinderToCell 
    {
        p1         (0.5 0.5 0);
        p2         (0.5 0.5 0.1);
        radius     0.05;
        fieldValues
        (
            volScalarFieldValue rho 1e-8
        );
    }
);
```

メッシュ作成からsetFieldsまでの実行は以下
```
blockMesh
setFields
```
## controlDictの変更
application名を変更。結果の保存感覚も変更。定常計算なので１ステップで計算が終了するため。
```
application     myPotentialFoam;   // icoFoam => myPotentialFoam

startFrom       startTime;

startTime       0;

stopAt          endTime;

endTime         0.5;

deltaT          0.005;

writeControl    timeStep;

writeInterval   1;              // 20 => 1

purgeWrite      0;

writeFormat     ascii;

writePrecision  6;

writeCompression off;

timeFormat      general;

timePrecision   6;

runTimeModifiable true;
```
## fvSchemesの変更
laplacian(phi)の項を追加
```
laplacianSchemes
{
    default         Gauss linear orthogonal;
    laplacian(phi)  Gauss linear corrected;
}
```
## fvSolutionの変更
pの項をphiに変更してそれ以外を削除。
```
solvers
{
    phi
    {
        solver          PCG;
        preconditioner  DIC;
        tolerance       1e-06;
        relTol          0.05;
    }
}
```

## caseの実行
```
myPoisonEqSolver
paraFoam
```
