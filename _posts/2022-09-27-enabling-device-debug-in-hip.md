---
layout: post
title: "Enabling GPU device debugging in HIP"
tags:
- GPU
- CUDA
- HIP
thumbnail_path: blog/2022-09-27-enabling-device-debug-in-hip/roc-gdb-logo.png
---

In order to debug a GPU kernel with `cuda-gdb`, we add `-G -O0` to nvcc command line, which in case of CMake would be:

```
enable_language(CUDA)
...
set(CMAKE_CUDA_FLAGS_DEBUG "-G -O0" ${CMAKE_CUDA_FLAGS_DEBUG})
```

In HIP it is done differently, because the compiler is a Clang flavor, and inherits its behavior. The HIP frontend compiler invokes backend compilers (which are also clang++) by itself. Therefore, the frontend clang++ needs -Xclang to forward the options down to the backends:

```
set(CMAKE_HIP_FLAGS_DEBUG "-ggdb -fstandalone-debug -Xclang -O0 -Xclang -gcodeview" ${CMAKE_HIP_FLAGS})
```

Note the debugging must always be performed with [roc-gdb](https://rocmdocs.amd.com/en/latest/ROCm_Tools/ROCgdb.html), even for the host code, because HIP's Clang and GCC these days tend to use incompatible debugging formats. As a result, you should get a beautiful debuggable code in TUI:

![alt text](\assets\img\blog\2022-09-27-enabling-device-debug-in-hip\roc-gdb.png)
