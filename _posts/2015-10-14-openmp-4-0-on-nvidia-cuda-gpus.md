---
layout: post
title: "OpenMP 4.0 on NVIDIA CUDA GPUs"
tags:
  - Software Engineering
thumbnail_path: blog/2015-10-14-openmp-4-0-on-nvidia-cuda-gpus/openmp4_nvidia.png
---

Multiple presentations about OpenMP 4.0 support on NVIDIA GPUs date back to 2012. There is however still very limited OpenMP 4.0 production-ready tools availability for NVIDIA devices: Intel's compilers are Xeon Phi only, PGI and Cray offer only OpenACC, GCC support is only in plans. Fortunately, a usable compiler could be built on top of LLVM and Clang. In this blog post we provide complete instructions to build OpenMP 4.0 compiler with NVIDIA support on Linux. Note these instructions could be especially useful for building OpenMP 4.0 compiler on platforms where OpenACC is not available, e.g. Jetson TK1.

![alt text](\assets\img\blog\2015-10-14-openmp-4-0-on-nvidia-cuda-gpus\openmp4_nvidia.png "Logo Title Text 1")

Create working directory:

```
$ mkdir -p $HOME/forge/openmp4
$ cd $HOME/forge/openmp4
```

Clone LLVM & Compiler Runtime & Clang sources:

```
$ git clone https://github.com/clang-omp/llvm_trunk llvm
$ git clone https://github.com/clang-omp/compiler-rt_trunk llvm/projects/compiler-rt
$ git clone https://github.com/clang-omp/clang_trunk llvm/tools/clang
```

Build & install Clang:

```
$ cd llvm/
$ mkdir build
$ cd build/
$ cmake -DCMAKE_INSTALL_PREFIX=$HOME/forge/openmp4/llvm/install ..
$ make -j12
$ make install
$ export PATH=$PATH:$HOME/forge/openmp4/llvm/install/bin/
$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/forge/openmp4/llvm/install/lib
```

Clone, build & install OpenMP runtime:

```
$ cd $HOME/forge/openmp4
$ git clone http://llvm.org/git/openmp.git
$ cd openmp/runtime/
$ mkdir build
$ cd build/
$ cmake -DCMAKE_INSTALL_PREFIX=$HOME/forge/openmp4/llvm/install ..
$ make -j12
$ make install
```

Clone, build & install OpenMP target backends (nvptx backend will be configured to support Compute Capabilities 30 and 35):

```
$ cd $HOME/forge/openmp4
$ git clone https://github.com/clang-omp/libomptarget.git
$ cd libomptarget
$ mkdir build
$ cd build
$ cmake -DCMAKE_INSTALL_PREFIX=$HOME/forge/openmp4/llvm/install -DOMPTARGET_NVPTX_SM=30,35 ..
$ make -j12
$ cp -rf lib/libomptarget* $HOME/forge/openmp4/llvm/install/lib/
```

Checkout the sample program:

```
$ cd $HOME/forge/openmp4
$ cat example.c
```

```
#include <malloc.h>
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char* argv[])
{
    if (argc != 2)
    {
        printf("Usage: %s \n", argv[0]);
        return 0;
    }

    int n = atoi(argv[1]);

    double* x = (double*)malloc(sizeof(double) * n);
    double* y = (double*)malloc(sizeof(double) * n);

    double idrandmax = 1.0 / RAND_MAX;
    double a = idrandmax * rand();
    for (int i = 0; i < n; i++)
    {
        x[i] = idrandmax * rand();
        y[i] = idrandmax * rand();
    }

    #pragma omp target data map(tofrom: x[0:n],y[0:n])
    {
        #pragma omp target
        #pragma omp for
        for (int i = 0; i < n; i++)
            y[i] += a * x[i];
    }

    double avg = 0.0, min = y[0], max = y[0];
    for (int i = 0; i < n; i++)
    {
        avg += y[i];
        if (y[i] > max) max = y[i];
        if (y[i] < min) min = y[i];
    }

    printf("min = %f, max = %f, avg = %f\n", min, max, avg / n);

    free(x);
    free(y);

    return 0;
}
```

```
$ cat makefile
```

```
all: example

example: example.c
    LIBRARY_PATH=$(shell dirname $(shell which clang-3.8))/../lib clang-3.8 -fopenmp -omptargets=nvptx64sm_30-nvidia-linux -g -O3 -std=c99 $< -o $@

clean:
    rm -rf example
```

Attach executable to nvprof to ensure CUDA kernels are launched on GPU in OpenMP backend:

```
$ make
LIBRARY_PATH=$HOME/forge/openmp4/llvm/install/bin/../lib clang-3.8 -fopenmp -omptargets=nvptx64sm_30-nvidia-linux -g -O3 -std=c99 example.c -o example
ptxas warning : Too big maxrregcount value specified 64, will be ignored

$ nvprof ./example 1024
==29138== NVPROF is profiling process 29138, command: ./example 1024
min = 0.025126, max = 1.773771, avg = 0.922563
==29138== Profiling application: ./example 1024
==29138== Profiling result:
Time(%)      Time     Calls       Avg       Min       Max  Name
 99.05%  3.3133ms         1  3.3133ms  3.3133ms  3.3133ms  __omptgt__0_27a163e_801_
  0.54%  18.047us         5  3.6090us  2.5600us  5.3430us  [CUDA memcpy DtoH]
  0.41%  13.568us         7  1.9380us  1.0880us  3.6160us  [CUDA memcpy HtoD]
```

Verify results against simple serial version:

```
$ gcc -std=c99 example.c -o example_cpu
$ ./example_cpu 1024
min = 0.025126, max = 1.773771, avg = 0.922563
```

References:

[http://clang-omp.github.io/]()

[https://github.com/clang-omp/libomptarget]()

[https://www.ibm.com/developerworks/community/blogs/8e0d7b52-b996-424b-bb33-345205594e0d?lang=en]()

**Update**: As [pointed out](https://www.linkedin.com/grp/post/1956294-6060039122796449794) by Dr. Maxime Hugues, Cray supports OpenMP 4.0 for NVIDIA GPUs starting from version 8.4.0.
