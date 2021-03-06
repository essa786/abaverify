# abaverify
A python package built on [`unittest`](https://docs.python.org/2.7/library/unittest.html) for running verification tests on Abaqus user subroutines. Basic familiarity with unittest and Abaqus user subroutine development is assumed in this readme.

This software may be used, reproduced, and provided to others only as permitted under the terms of the agreement under which it was acquired from the U.S. Government. Neither title to, nor ownership of, the software is hereby transferred. This notice shall remain on all copies of the software.

Copyright 2016 United States Government as represented by the Administrator of the National Aeronautics and Space Administration. No copyright is claimed in the United States under Title 17, U.S. Code. All Other Rights Reserved.

For any questions, please contact the developers:
- Andrew Bergan | [andrew.c.bergan@nasa.gov](mailto:andrew.c.bergan@nasa.gov) | (W) 757-864-3744
- Frank Leone   | [frank.a.leone@nasa.gov](mailto:frank.a.leone@nasa.gov)     | (W) 757-864-3050

## Getting-Started
This package assumes that you have `python 2.7` and `git` installed. This packaged is designed for Abaqus 2016 and it has been used successfully with v6.14; it may or may not work with older versions. It also assumes that you have an Abaqus user subroutine in a git repository with a minimum directory structure as shown here:
```
.
    .git/
    for/
        usub.for
    tests/
        testOutput/
        abaqus_v6.env
        test_model1.inp
        test_model1_expected.py
        ...
        test_runner.py
    .gitignore
```

The user subroutine is stored in the `for/` directory and the verification tests are stored in the `<your_userSubroutine_repo_dir>/tests/` directory.

### Install `abaverify`
To install `abaverify`, simply clone it to your subroutine's `tests/` directory.

The remainder of this section describes how to build your own tests using `abaverify` (e.g., what goes inside the `test_model1.inp`, `test_model1_expected.py`, and `test_runner.py`) files. For a working example, checkout the sample verification test in the `abaverify/tests/tests/` directory. You can run the sample test with the command `python test_runner.py` from the `abaverify/tests/tests/` directory. Note, the default environment file (`abaverify/tests/tests`) is formatted for windows; linux users will need to modify the default environment file to the linux format.

### Create `.inp` and `.py` files for each test model
A model file (`*.inp` or `*.py`) and corresponding results file (`*_expected.py`) with the same name must be created for each test case. These files are placed in the `tests/` directory. The model file is a typical Abaqus input deck or python script, so no detailed discussion is provided here (any Abaqus model should work). When tests are executed (with the command `python test_runner.py`), the models in the `tests/` directory are run in Abaqus.

The `*_expected.py` file defines the assertions that are run on the `odb` output from the analysis. After each analysis is completed, the script `abaverify/processresults.py` is called to collect the quantities defined in the `*_expected.py` file. The `*_expected.py` file must contain a list called `"results"` that contains an object for each result of interest. A typical result quantity would be the maximum stress for the stress component `S11`. For example:
```
parameters = {
    "results":
        [
            # Simple example to find max value of state variable d2
            {
                "type": "max",                                      # Specifies criteria to apply to the output quantity
                "identifier":                                       # Identifies which output quantity to interrogate
                    {
                        "symbol": "SDV_CDM_d2",
                        "elset": "ALL",
                        "position": "Element 1 Int Point 1"
                    },
                "referenceValue": 0.0                               # This the value that the result from the model is compared against
            },
            {
            ...
            <Additional result object here>
            ...
            }
        ]
}   
```
The value found in the `odb` must match the reference value for the test to pass. In the case above, the test is simply to say that `SDV_CDM_d2` is always zero, since the range of `SDV_CDM_d2` happens to be between 0 and 1. Any history output quantity can be interrogated using one of the following criteria defined in the `type` field: `max`, `min`, `continuous`, `xy_infl_pt`, `disp_at_zero_y`, `log_stress_at_failure_init`, or `slope`. Here is a more complicated example:
```
parameters = {
    "results":
        [
            # More complicated case to find the slope of the stress strain curve within the interval 0.0001 < x < 0.005
            {
                "type": "slope",
                "step": "Step-1",                                   # By default the step is assumed to be the first step. Can specify any step with the step name
                "identifier": [                                     # The identifier here is an array since we are looking for the slope of a curve defined by x and y
                    { # x
                        "symbol": "LE11",
                        "elset": "ALL",
                        "position": "Element 1 Int Point 1"
                    },
                    { # y
                        "symbol": "S11",
                        "elset": "ALL",
                        "position": "Element 1 Int Point 1"
                    }
                ],
                "window": [0.0001, 0.005],                          # [min, max] in x        
                "referenceValue": 171420,                           # Reference value for E1
                "tolerance": 1714                                   # Require that the calculated value is within 1% of the reference value
            }
        ]
}
```
The results array can contain as many result objects as needed to verify that the model has performed as designed. Assertions are run on each result object and if any one fails, the test is marked as failed.

### Create a `test_runner.py` file
The file `sample_usage.py` gives an example of how you call your newly created tests. By convention, this file is named as `test_runner.py`. This file must include:
1. `import abaverify`.
2. classes that inherit `av.TestCase` and define functions beginning with `test` following the usage of `unittest`. See the `sample_usage.py` for an example.
3. call to `runTests()` which takes one argument: the relative path to your user subroutine (omit the `.f` or `.for` ending, the code automatically appends it).

## Running your tests
Before running tests, make sure you place an Abaqus environment file in your project's `tests/` directory. At a minimum, the environment file should include the options for compiling your subroutine. If you do not include your environment file, `abaverify` will give an error.

You can run your tests with the syntax defined by `unittest`. To run all tests, execute the following from the `tests` directory of your project:
```
tests $  python test_runner.py
```
All of the tests that have been implemented will be run. The last few lines of output from these commands indicate the number of tests run and `OK` if they are all successful.

To run a single test, add the class and test name. For example, for the input deck `test_CPS4R_tension.inp`, type:
```
tests $  python test_runner.py SingleElementTests.test_CPS4R_tension
```

Various command line options can be used as described below.

The option `-i` or equivalently `--interactive` can be specified to print the Abaqus log data to the terminal. For example:
```
tests $  python test_runner.py SingleElementTests.test_C3D8R_simpleShear12 --interactive
```

The option `-t` or equivalently `--time` can be specified to print the run times for the compiler, packager, and solver to the terminal. For example:
```
tests $  python test_runner.py SingleElementTests.test_C3D8R_simpleShear12 --timer
```

The option `-c` or equivalently `--preCompileCode` can be specified to use `abaqus make` to compile the code into a binary before running one or more tests. A function that compiles the code must be provided to `abaverify` as an argument to the `runTests` function call in `test_runner.py`. The `usub_lib` option must be defined in the environment file.

The option `-e` or equivalently `--useExistingBinaries` can be specified to reuse the most recent compiled version of the code.

The option `-r` or equivalently `--useExistingResults` can be specified to reuse the most recent test results. The net effect is that only the post-processing portion of the code is run, so you don't have to wait for the model to run just to debug a `_expected.py` file or `processresults.py`.

The option `-s` or equivalently `--specifyPathToSub` can be used to override the relative path to the user subroutine specified in the the call `av.runTests()` in your `test_runner.py` file.
