---
layout: post
title: "Use CUDA 7.0 NVRTC with Thrust"
tags:
  - Software Engineering
thumbnail_path: blog/2015-04-29-use-cuda-7-0-nvrtc-with-thrust/nvrtc.png
---

Rintime Compilation (NVRTC) introduced in CUDA 7.0 allows to dynamically compile CUDA kernels during program execution (see [example](http://docs.nvidia.com/cuda/nvrtc/index.html#example-saxpy)). This functionality allows to perform additional GPU code optimization/specialization, using runtime context, e.g. to substitute constant loop bounds and unroll loops, or to eliminate divergent branches known not to be visited. However, NVRTC is not fully equivalent to offline nvcc: it only compiles CUDA **device** code into PTX assembly. Thus, NVRTC is not directly usable with GPU-enabled frameworks that combine both host and device code, e.g. Thrust. In this demo we show how to use NVRTC to replace a certain device function in Thrust code.

![alt text](\assets\img\blog\2015-04-29-use-cuda-7-0-nvrtc-with-thrust\nvrtc.png "Logo Title Text 1")

First of all, let's prepare a sample Thrust transform:

```
#include
#include <thrust/for_each.h>
#include <thrust/device_vector.h>

__device__ void function(int x);

struct functor
{
    __device__ void operator()(int x)
    {
        function(x);
    }
};

void for_each()
{
    printf("for_each\n");

    thrust::device_vector d_vec(3);
    d_vec[0] = 0; d_vec[1] = 1; d_vec[2] = 2;
    thrust::for_each(d_vec.begin(), d_vec.end(), functor());
}
```

Here, we have thrust::for_each transform on vector elements, whose functor is calling external device function. For our initiali executable we will compile this code together with default function implementation:

```
__device__ void function(int x) { }
```

We will also compile the same "template" Thrust code without default function into relocatable PTX assembly:

```
functor.ptx: functor.cu
    nvcc -g -arch=sm_35 -rdc=true -ptx -c $< -o $@
```

The main program also contains the version of device function that shall dynamically replace the default one:

```
const char* functionSource = "                                  \n\
__device__ void function(int x)                                 \n\
{                                                               \n\
    printf(\"%d\\n\", x);                                       \n\
}                                                               \n";
```

We compile it using NVRTC and link against "template" Thrust code:

```
// Create an instance of nvrtcProgram with the SAXPY code string.
nvrtcProgram prog;
NVRTC_SAFE_CALL(nvrtcCreateProgram(&prog, functionSource, "function", 0, NULL, NULL));

// Compile the program for compute_35.
const char *opts[] = {"--gpu-architecture=compute_35", "-rdc=true" };
nvrtcResult compileResult = nvrtcCompileProgram(prog, 2, opts);

// Obtain compilation log from the program.
size_t logSize;
NVRTC_SAFE_CALL(nvrtcGetProgramLogSize(prog, &logSize));
if (logSize > 1)
{
    std::vector log(logSize);
    NVRTC_SAFE_CALL(nvrtcGetProgramLog(prog, &log[0]));
    std::cout << &log[0] << std::endl;
}
if (compileResult != NVRTC_SUCCESS)
    exit(1);

// Obtain PTX from the program.
size_t ptxSize;
NVRTC_SAFE_CALL(nvrtcGetPTXSize(prog, &ptxSize));
char *ptx = new char[ptxSize];
NVRTC_SAFE_CALL(nvrtcGetPTX(prog, ptx));

// Destroy the program.
NVRTC_SAFE_CALL(nvrtcDestroyProgram(&prog));

// Load precompiled relocatable source with call to external function
// and link it together with NVRTC-compiled function.
CUlinkState linker;
CUDA_SAFE_CALL(cuLinkCreate(0, NULL, NULL, &linker));
CUDA_SAFE_CALL(cuLinkAddFile(linker, CU_JIT_INPUT_PTX, "functor.ptx", 0, NULL, NULL));
CUDA_SAFE_CALL(cuLinkAddData(linker, CU_JIT_INPUT_PTX, (void*)ptx, ptxSize, "function.ptx", 0, NULL, NULL));
void* cubin;
CUDA_SAFE_CALL(cuLinkComplete(linker, &cubin, NULL));
CUDA_SAFE_CALL(cuModuleLoadDataEx(&module, cubin, 0, NULL, NULL));
CUDA_SAFE_CALL(cuLinkDestroy(linker));

As result, CUDA module contains binary code for exactly the same Thrust source we have in initial executable, combined with the new device function. Remaining tricky part is to redirect original Thrust code kernel launches to our new module. This is done using CUDA runtime hooks:

?
extern "C" void __cudaRegisterFunction(void** fatCubinHandle,
    const char* hostFun, char* deviceFun, const char* deviceName,
    int thread_limit, uint3* tid, uint3* bid, dim3* bDim, dim3* gDim, int* wSize)
{
    ...
}

cudaError_t cudaConfigureCall(dim3 gridDim, dim3 blockDim, size_t sharedMem, cudaStream_t stream)
{
    ...
}

cudaError_t cudaSetupArgument(const void *arg, size_t size, size_t offset)
{
    ...
}

cudaError_t cudaLaunch(const void* entry)
{
    ...
}
```

Finally, compiled Thrust executable runs NVRTC-generated device function:

```
$ make
nvcc -g -arch=sm_35 -rdc=true -ptx -c functor.cu -o functor.ptx
nvcc -g -arch=sm_35 nvrtc.cu -o nvrtc -cudart=shared -lnvrtc -lcuda
$ ./nvrtc
for_each
0
1
2
```

Download full demo source code [here](https://parallel-computing.pro/articles/nvrtc/nvrtc.zip)
