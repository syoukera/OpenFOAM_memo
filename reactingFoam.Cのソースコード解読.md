# reactingFoam.Cのソースコード解読
## ヘッダファイル
```
#include "fvCFD.H"
#include "turbulentFluidThermoModel.H"
#include "psiReactionThermo.H"
#include "CombustionModel.H"
#include "multivariateScheme.H"
#include "pimpleControl.H"
#include "pressureControl.H"
#include "fvOptions.H"
#include "localEulerDdtScheme.H"
#include "fvcSmooth.H"
```
|ヘッダファイル|説明|
|:-----:|:-----:|
|fvCFD.H|CFDを行うためのヘッダファイル詰め合わせ|
|turbulentFluidThermoModel.H|乱流モデル|
|psiReactionThermo.H|
|CombustionModel.H|燃焼モデル|
|multivariateScheme.H|他変数の離散化スキーム|
|pimpleControl.H|PIMPLEでの収束についての情報取得|
|pressureControl.H|閉鎖系での圧力制御|
|fvOptions.H|有限体積法のオプション|
|localEulerDdtScheme.H|1次のオイラー法ddtスキーム|
|fvcSmooth.H|スムーススプデッドとスウィープ|
## mainその１
```
argList::addNote
    (
        "Solver for combustion with chemical reactions"
    );

    #include "postProcess.H"

    #include "addCheckCaseOptions.H"
    #include "setRootCaseLists.H"
    #include "createTime.H"
    #include "createMesh.H"
    #include "createControl.H"
    #include "createTimeControls.H"
    #include "initContinuityErrs.H"
    #include "createFields.H"
    #include "createFieldRefs.H"
```
## mainその２
```
    turbulence->validate();

    if (!LTS)
    {
        #include "compressibleCourantNo.H"
        #include "setInitialDeltaT.H"
    }
```
## mainその３
```
    Info<< "\nStarting time loop\n" << endl;

    while (runTime.run())
    {
    \\ 以下のコード
    }
```
## main{ while{その１} }
```
        #include "readTimeControls.H"

        if (LTS)
        {
            #include "setRDeltaT.H"
        }
        else
        {
            #include "compressibleCourantNo.H"
            #include "setDeltaT.H"
        }

        ++runTime;
        
        Info<< "Time = " << runTime.timeName() << nl << endl;
```
## main{ while{その２} }
```
        #include "rhoEqn.H"

        while (pimple.loop())
        {
        //　以下のコード 
        }
```
## main{ while{ while{ その１} } }
```
            #include "UEqn.H"
            #include "YEqn.H"
            #include "EEqn.H"

            // --- Pressure corrector loop
            while (pimple.correct())
            {
                if (pimple.consistent())
                {
                    #include "pcEqn.H"
                }
                else
                {
                    #include "pEqn.H"
                }
            }
```
## main{ while{ while{その２} } }
```
            if (pimple.turbCorr())
            {
                turbulence->correct();
            }
        } // while ループ終わり
```
## main{ while { その３ } } }
```
        rho = thermo.rho();

        runTime.write();

        runTime.printExecutionTime(Info);
    } // while ループ終わり
```
## main{ その４ }
```
    Info<< "End\n" << endl;

    return 0;
} // main 終わり
```

### 作成途中
