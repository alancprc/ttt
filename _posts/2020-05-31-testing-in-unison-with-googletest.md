---
layout:     post
title:      testing in Unison with googletest
categories: c++
tags:       Unison googletest
---

* 目录
{:toc}

Suppose we've written an application library in [Unison](https://www.cohu.com/semiconductor-ate-software), We could do some test to make sure that it works as expected.

In this article, we'll walk through writing a simple testcase in Unison with
googletest.

## What is googletest?
* Google's C++ test framework. [https://github.com/google/googletest](https://github.com/google/googletest)
* You'll find the document here [https://github.com/google/googletest/tree/master/googletest/docs](https://github.com/google/googletest/tree/master/googletest/docs).
* It will be a good start to read this [https://github.com/google/googletest/blob/master/googletest/docs/primer.md](https://github.com/google/googletest/blob/master/googletest/docs/primer.md).

## Build googletest as a shared library
First of all, to work in Unison, we need to build googletest into a shared library.

### Choose the right version of googletest
Starting from version v1.10.0, googletest requires the compiler support C++11 standard, while version 1.8.x is the last releases that works with pre-C++11 compilers.

* For CentOS 7, the gcc (version 4.8.x) fully supports C++11, so the latest version would be prefered.
* For CentOS 6, the gcc (version 4.4.x) support only part of C++11, so only v1.8.x is available. Still, with the experimental C++0x, part of C++11 features is provided.

### Get source code of googletest
You can clone the git repository, or download the source tarball.
In this example, release-v1.10.0 will be used. we'll build it under `~/build-googletest`
* Clone git repository
    ```bash
    mkdir ~/build-googletest
    cd ~/build-googletest
    git clone https://github.com/google/googletest.git
    cd googletest
    # list all released version
    git tag
    # possible output:
        release-1.0.0
        release-1.0.1
        release-1.1.0
        release-1.10.0
        release-1.2.0
        release-1.2.1
        release-1.3.0
        release-1.4.0
        release-1.5.0
        release-1.6.0
        release-1.7.0
        release-1.8.0
        release-1.8.1
        v1.10.x # unreleased
    # we'll check out the latest release
    git checkout release-1.10.0
    ```
* Download source tarball
    ```bash
    mkdir ~/build-googletest
    cd ~/build-googletest
    wget https://github.com/google/googletest/archive/release-1.10.0.tar.gz
    tar xzf googletest-release-1.10.0.tar.gz
    ```
### Build shared library
I've created a github repository [alancprc/google-test-build-as-shared-lib](https://github.com/alancprc/google-test-build-as-shared-lib), containing the Makefile to build googletest as a shared library, you should get the `Makefile` before going on.
* Put the `Makefile` under `~/build-googletest`
* The default option in Makefile is `-m64`, which tells the compiler to generate code for a 64-bit environment. `-m64` is for U1709+, you'll need to replace `-m64` with `-m32` for Unison 4.x
* Run `make` to build
* If everything goes well, you'll get following .so files
    ```
    libgtest.so
    libgtest_main.so
    libgmock.so
    libgmock_main.so
    ```
    and the following output
    ```
    Running main() from gmock_main.cc
    [==========] Running 6 tests from 2 test cases.
    [----------] Global test environment set-up.
    [----------] 3 tests from FactorialTest
    [ RUN      ] FactorialTest.Negative
    [       OK ] FactorialTest.Negative (0 ms)
    [ RUN      ] FactorialTest.Zero
    [       OK ] FactorialTest.Zero (0 ms)
    [ RUN      ] FactorialTest.Positive
    [       OK ] FactorialTest.Positive (0 ms)
    [----------] 3 tests from FactorialTest (0 ms total)
    
    [----------] 3 tests from IsPrimeTest
    [ RUN      ] IsPrimeTest.Negative
    [       OK ] IsPrimeTest.Negative (0 ms)
    [ RUN      ] IsPrimeTest.Trivial
    [       OK ] IsPrimeTest.Trivial (0 ms)
    [ RUN      ] IsPrimeTest.Positive
    [       OK ] IsPrimeTest.Positive (0 ms)
    [----------] 3 tests from IsPrimeTest (0 ms total)
    
    [----------] Global test environment tear-down
    [==========] 6 tests from 2 test cases ran. (0 ms total)
    [  PASSED  ] 6 tests.
    ```

## Install googletest to system directories
* Put the generated shared libraries into `/usr/lib64`, or `/usr/lib` if `-m32` option is set. root permission is required.
    ```
    cp -at /usr/lib64/ libgtest*.so libgmock*.so
    ```
* put the header files to /usr/local/include
    ```
    cp -at /usr/local/include ./googletest/googletest/include/gtest ./googletest/googletest/include/gmock
    ```

## Use googletest in Unison application library
### Library to test
In this example, we create a Unison application library called UnisonUtil, containing files `unison-util.h` and `unison-util.cpp`, providing a simple function ` void removeTrailNewLine(StringS &str)`. The source code is as follows:

* unison-util.h
    ```cpp
    #pragma once
    #include <Unison.h>
    
    namespace util {
    
    /** @brief remove trailing new line '\n' in string.*/
    void removeTrailNewLine(StringS &str);
    
    }  // namespace util
    ```
* unison-util.cpp
    ```cpp
    #include "unison-util.h"
    
    namespace util {
    
    void removeTrailNewLine(StringS &str)
    {
      int len = str.Length();
      if (len and str[len - 1] == '\n') {
        str.Erase(len - 1, 1);
      }
    }
    
    }  // namespace util
    ```

### Create UnitTest application library
1. Create an application library UnitTest in Unison TestTool.
2. Add a source file unittest.cpp, with following code
    ```cpp
    #include <gmock/gmock.h>
    #include "unison-util.h"
    
    using namespace std;
    
    TMResultM StartGTest()
    {
     int argc = 1;
     const char* argv[] = {"a.out"};
    
     std::cout << "\n\n\nRunning main() from unittest.cpp\n";
     testing::InitGoogleMock(&argc, const_cast<char**>(argv));
     int result = RUN_ALL_TESTS();
     return result == 0 ? TM_PASS : TM_FAIL;
    }
    
    TEST(UtilFuncTest, removeTrailNewLineTest)
    {
     StringS str = "hello world\n";
     util::removeTrailNewLine(str);
     util::removeTrailNewLine(str);
     EXPECT_STREQ(str, "hello world");
    }
    ```

### Set up the UnitTest library

* Add following to Compiler Flags
    ```
    -pthread
    -DGTEST_LINKED_AS_SHARED_LIBRARY=1
    -fno-gnu-unique
    ```
* Add following to Linker Flags
    ```
    -lpthread
    -lgmock -lgtest
    ```

* Add include path for UnisonUtil and googletest
    ```
    ./                      # include path for UnisonUtil
    /usr/local/include      # include path for googletest
    ```
* Add library UnisonUtil as dependency

### Build and start test
* build UnitTest library
* Create a TestGroup `tgStartGTest`, which calls `StartGTest()` function
* Execute the TestGroup
    you should see the following output in dataviewer
    ```
    Running main() from unittest.cpp
    [==========] Running 1 test from 1 test suite.
    [----------] Global test environment set-up.
    [----------] 1 test from UtilFuncTest
    [ RUN      ] UtilFuncTest.removeTrailNewLineTest
    [       OK ] UtilFuncTest.removeTrailNewLineTest (0 ms)
    [----------] 1 test from UtilFuncTest (0 ms total)
    
    [----------] Global test environment tear-down
    [==========] 1 test from 1 test suite ran. (0 ms total)
    [ PASSED ] 1 test.
    ```

## Known issue
* Unable to close the dynamic-linked library file
    ```
    << __WARNING 1:___________________________________

    In utlrt-aliang_sim (DynamicLink_Method), 4gl/class/dll_method/DLL_Installation.cxx, line 897, at Sun May 31 12:06:55 2020

    Warning: Uninstalling Dynamic-Link Method:

    File: /home/aliang/working/unison-util/x86_64_linux_3.10.0/libUnitTest.un.so

    Unable to close the dynamic-linked library file
        Please check for any static variables in inlined functions and move them out so they are not inlined. The test program has to be unloaded and loaded back after changing it. Running 'readelf -Ws libUnitTest.un.so | grep UNIQUE | c++filt' will help to find what is causing the error.
    User file: 4gl/class/dll_method/DLL_Installation.cxx, Line: 897, Statement: DynamicLink_Method
    ```
    * This error may occur on CentOS 7 or CentOS 6 with an updated gcc version. To solve this error, add compiler flag `-fno-gnu-unique` to the application library.


## Why not build googletest into a Unison application library?  
Though installing googletest into system directory is disencouraged by googletest document, I found it necessary to use it in Unison.  
I've tried to build googletest into a Unison application library, but when the library gets re-compiled, the tests get duplicated, which is really annoying.  
For example, if the above test uses a googletest as a Unison application library, it'll become something like following after re-compiling UnisonUtil library.
```
Running main() from unittest.cpp
[==========] Running 2 tests from 1 test suites.
[----------] Global test environment set-up.
[----------] 2 tests from UtilFuncTest
[ RUN      ] UtilFuncTest.removeTrailNewLineTest
[       OK ] UtilFuncTest.removeTrailNewLineTest (0 ms)
[ RUN      ] UtilFuncTest.removeTrailNewLineTest
[       OK ] UtilFuncTest.removeTrailNewLineTest (0 ms)
[----------] 2 tests from UtilFuncTest (0 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 1 test suites ran. (0 ms total)
[  PASSED  ] 2 tests.
```
if the UnisonUtil library gets another re-compiling, the output becomes as follows.
```
Running main() from unittest.cpp
[==========] Running 3 tests from 1 test suites.
[----------] Global test environment set-up.
[----------] 3 tests from UtilFuncTest
[ RUN      ] UtilFuncTest.removeTrailNewLineTest
[       OK ] UtilFuncTest.removeTrailNewLineTest (0 ms)
[ RUN      ] UtilFuncTest.removeTrailNewLineTest
[       OK ] UtilFuncTest.removeTrailNewLineTest (0 ms)
[ RUN      ] UtilFuncTest.removeTrailNewLineTest
[       OK ] UtilFuncTest.removeTrailNewLineTest (0 ms)
[----------] 3 tests from UtilFuncTest (0 ms total)

[----------] Global test environment tear-down
[==========] 3 tests from 1 test suites ran. (0 ms total)
[  PASSED  ] 3 tests.
```

Original article <https://alancprc.github.io/c++/2020/05/31/testing-in-unison-with-googletest.html>
Link of this article <https://alancprc.github.io/c++/2020/05/31/testing-in-unison-with-googletest.html>
Please indicate the source when reprint
