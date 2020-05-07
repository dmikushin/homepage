---
layout: post
title: "Using CUDA device functions from OpenACC"
tags:
  - Software Engineering
thumbnail_path: blog/2016-09-16-using-cuda-device-functions-from-openacc/openacc_device_function.jpg
---

OpenACC enables rapid transition of serial C/C++/Fortran into GPU-enabled parallel code. However, due to high-level nature, OpenACC does not offer access to GPU-specific features useful for debugging, optimization and other purposes. In this article we demonstrate how to call CUDA device functions from within OpenACC kernels by two examples: GPU compute grid retrieval and printf.

![alt text](\assets\img\blog\2016-09-16-using-cuda-device-functions-from-openacc\openacc_device_function.jpg "Logo Title Text 1")

In OpenACC source file make forward declarations of our CUDA device functions:

```C++
// Declaration of 3-integer structure, which is built-in
// in CUDA, but not in C/OpenACC.
typedef struct { int x, y, z; } int3;

// Return thread 3D index.
#pragma acc routine
int3 acc_get_thread_idx();

// Return block 3D index.
#pragma acc routine
int3 acc_get_block_idx();

// Print values from within the OpenACC parallel for loop.
#pragma acc routine
void print(int3 thread, int3 block, int i, float a, float b, float c);
```

Now, we can call these functions from the OpenACC parallel loop:

```C++
#pragma acc parallel for private(i)
for (i = 0; i < n; i++)
{
  int3 thread = acc_get_thread_idx();
  int3 block = acc_get_block_idx();
  c[i] = a[i] + b[i];
  print(thread, block, i, a[i], b[i], c[i]);
}
```

CUDA device functions are to be defined in separate compute_grid.cu and print.cu files:

```C++

// Return thread 3D index.
extern "C" __device__ int3 acc_get_thread_idx()
{
    int3 result;
    result.x = threadIdx.x;
    result.y = threadIdx.y;
    result.z = threadIdx.z;

    return result;
}

// Return block 3D index.
extern "C" __device__ int3 acc_get_block_idx()
{
    int3 result;
    result.x = blockIdx.x;
    result.y = blockIdx.y;
    result.z = blockIdx.z;

    return result;
}
```

```C++

#include <cstdio>

// Print values from within the OpenACC parallel for loop.
extern "C" __device__ void print(int3 thread, int3 block, int i, float a, float b, float c)
{
    printf("block: (%d, %d, %d), thread: (%d, %d, %d) :: i = %d, a = %f, b = %f, c = %f\n",
        block.x, block.y, block.z, thread.x, thread.y, thread.z, i, a, b, c);
}
```

The last building block is a Makefile to compile and link this all together. Note the "rdc" flag needed to create linkable CUDA device code:

```
CC = pgcc
CFLAGS = -acc -g -O3 -ta=nvidia:cc30 -Mcuda=rdc
NVCC = /opt/cuda7.0/bin/nvcc
NVCCFLAGS = -rdc=true -arch=sm_30

all: main

main: main.o compute_grid.o print.o
    $(CC) $(CFLAGS) $^ -o $@ -L$(shell dirname $(shell which $(NVCC)))/../lib64 -lcudart

main.o: main.c
    $(CC) $(CFLAGS) -c $< -o $@

compute_grid.o: compute_grid.cu
    $(NVCC) $(NVCCFLAGS) -c $< -o $@

print.o: print.cu
    $(NVCC) $(NVCCFLAGS) -c $< -o $@

clean:
    rm -rf main *.o
```

Now, our test program is capable of showing compute grid config and printing from withing the OpenACC code, which is not directly supported by OpenACC itself.

Small problem (n = 128) takes only one block:

```
$ ./main 128
...
block: (0, 0, 0), thread: (62, 0, 0) :: i = 62, a = 62.000000, b = 124.000000, c = 186.000000
...
block: (0, 0, 0), thread: (127, 0, 0) :: i = 127, a = 127.000000, b = 254.000000, c = 381.000000

Larger problem (n = 128) takes 2 blocks (OpenACC obviously uses (128, 1, 1) blocks):
```

Larger problem (n = 256) takes 2 blocks (OpenACC obviously uses (128, 1, 1) blocks):

```
$ ./main 256
...
block: (0, 0, 0), thread: (62, 0, 0) :: i = 62, a = 62.000000, b = 124.000000, c = 186.000000
...
block: (0, 0, 0), thread: (127, 0, 0) :: i = 127, a = 127.000000, b = 254.000000, c = 381.000000
...
block: (1, 0, 0), thread: (62, 0, 0) :: i = 190, a = 190.000000, b = 380.000000, c = 570.000000
...
block: (1, 0, 0), thread: (127, 0, 0) :: i = 255, a = 255.000000, b = 510.000000, c = 765.000000
```
