---
layout: post
current: post
cover:  assets/images/pytorch-cpp.png
navigation: True
title: "PyTorch Setup (C++17, zapcc, QtCreator, Debian, user-space)"
date: 2019-10-24 01:00:00
tags: [c++, machine learning]
class: post-template
subclass: 'post'
author: philip
excerpt: 'Harder than it should be.'
---

This post documents my [PyTorch](https://pytorch.org) C++ setup.
It is intended as a brief how-to.


## Goals

* Works with C++17 code (no pre-C++11 ABI)
* Works with the [zapcc](https://github.com/yrnkrn/zapcc) compiler (personal favorite)
* Works with QtCreator (currently my favored IDE on linux)
* Works with Debian without sudo rights (work constraint)
* Works with CUDA (only realistic way to train larger networks)


## Steps

(All versions were current at the time of writing)


### Installing PyTorch C++

* go to [https://pytorch.org/](https://pytorch.org/)
* scroll down to configure download
* select:
    * Build: `Stable (1.3)`
    * OS: `Linux`
    * Package: `LibTorch`
    * Language: `C++`
    * CUDA: `10.1`
* download with `cxx11 ABI` (*Important!*)

Example link: [https://download.pytorch.org/libtorch/cu101/libtorch-cxx11-abi-shared-with-deps-1.3.0.zip](https://download.pytorch.org/libtorch/cu101/libtorch-cxx11-abi-shared-with-deps-1.3.0.zip)


### Installing Cuda Toolkit 10.1

* go to [https://developer.nvidia.com/cuda-downloads](https://developer.nvidia.com/cuda-downloads)
* select `Linux -> x86_64 -> Ubuntu -> 18.04 -> runfile (local)` 

  (should give you `wget <link>`)
* instead of executing, extract it via `xyz.run --tar mxvf`
* run the `cuda-installer` in the extracted folder
    * do not install driver, samples, demo suite
    * go to `Options -> Toolkit Options`
        * change Toolkit Install Path to `/local/something/cuda-10.1`
        * do not create links
        * do not install manpage
    * go to `Options -> Library install path`
        * change to `/local/something/cuda-10.1`

### Installing CuDNN

* go to [https://developer.nvidia.com/cudnn](https://developer.nvidia.com/cudnn)
* download CuDNN
* extract file
* copy `lib64` and `include` folders to `/local/something/cuda-10.1/`


### CMake

#### Fix `Caffe2Targets.cmake`

Caffe2 contains hard-coded CUDA paths that are wrong in our installation.
Search `Caffe2Targets.cmake` for `lib64/libcudart.so` and replace all absolute CUDA paths by your local installation.


#### PyTorch

I chose to explicitly provide a hint to the pytorch path.

```cmake
# pytorch
find_package(Torch REQUIRED
    # path containing bin/lib/include/share
    PATHS "/path/to/libtorch/"
)
```

*NOTE*: the cmake variable `TORCH_LIBRARY` was set to a wrong value anyways.
In QtCreator `Projects -> Build -> CMake -> TORCH_LIBRARY` change the value to `/path/to/libtorch/lib/libtorch.so`.

*NOTE*: the cmake variable `CUDA_TOOLKIT_ROOT_DIR` was missing in my case.
In QtCreator `Projects -> Build -> CMake -> CUDA_TOOLKIT_ROOT_DIR` change the value to `/path/to/cuda-10.1/`.


#### OpenMP

When using zapcc, openmp is not found by default.
We fix this by providing a clang-7 openmp (which zapcc is based on).

```cmake
# OpenMP
find_package(OpenMP)
if (OPENMP_CXX_FOUND)
    set(OPENMP_TARGET OpenMP::OpenMP_CXX)
elseif(NOT MSVC)
    add_library(openmp INTERFACE)
    target_link_libraries(openmp INTERFACE /usr/lib/llvm-7/lib/libomp.so)
    target_include_directories(openmp INTERFACE /usr/lib/llvm-7/include/openmp)
    set(OPENMP_TARGET openmp)
else()
    set(OPENMP_TARGET "")
endif()
```

#### Our Project

```cmake
target_link_libraries(OurProjectName PUBLIC
    # ... other dependencies
    ${TORCH_LIBRARIES}
    ${OPENMP_TARGET}
)

if (NOT MSVC)
    target_compile_options(OurProjectName PUBLIC
        # ... other options
        -fopenmp
    )
endif()
```

## Running The Program

If the `LD_LIBRARY_PATH` is not set to `/path/to/cuda-10.1/lib64`, some libraries will not be found when running the program.

In the QtCreator, this can be added in `Projects -> Run -> Run Environment -> Add`.

The example code at [https://pytorch.org/cppdocs/frontend.html](https://pytorch.org/cppdocs/frontend.html) should work now.
