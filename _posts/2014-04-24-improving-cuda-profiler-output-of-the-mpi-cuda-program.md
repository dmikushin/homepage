---
layout: post
title: "Improving CUDA profiler output of the MPI-CUDA program"
tags:
- Software Engineering
thumbnail_path: blog/2014-04-24-improving-cuda-profiler-output-of-the-mpi-cuda-program/mpi-cuda.jpg
---

Consider we need to profile the following MPI-CUDA program on GPU cluster. The most obvious way to profile this code on console-only cluster would be to invoke the profiler inside the mpirun command:

```c++
#include <stdio.h>
#include <stdlib.h>

#define CUDA_ERR_CHECK(x) \
	do { cudaError_t err = x; if (err != cudaSuccess) {          \
		fprintf (stderr, "Error \"%s\" at %s:%d \n",         \
		 cudaGetErrorString(err),                            \
		__FILE__, __LINE__); exit(-1);                       \
	}} while (0);

__global__ void gpu_kernel(double* gpu_data)
{
	int idx = threadIdx.x + blockDim.x * blockIdx.x;
	gpu_data[idx] += 1.0f;
}

int profile_gpu(int rank)
{
	int nthreads = 4, nblocks = 4;
	int n = nthreads * nblocks;
	double* host_data = (double*)malloc(n * sizeof(double));
	for (int i = 0; i < n; i++)
		host_data[i] = i;
	double* gpu_data = NULL;
	CUDA_ERR_CHECK( cudaMalloc(&gpu_data, n * sizeof(double)) );
	CUDA_ERR_CHECK( cudaMemcpy(gpu_data, host_data, n * sizeof(double),
		cudaMemcpyHostToDevice) );

	gpu_kernel<<<nblocks, nthreads>>>(gpu_data);
	CUDA_ERR_CHECK( cudaGetLastError() );
	CUDA_ERR_CHECK( cudaDeviceSynchronize() );

	return 0;
}
```

```
mpirun -np 4 nvprof --metrics flops_dp ./nvtest
```

Here we collect the double precision FLOPS number on GPU using profiler. Unfortunately due to non-buffered output nvprof results will be mixed and will have no information about MPI ranks:

```
$ mpirun -np 4 nvprof --metrics flops_dp ./nvtest 
==28738== NVPROF is profiling process 28738, command: ./nvtest
==28740== NVPROF is profiling process 28740, command: ./nvtest
==28736== NVPROF is profiling process 28736, command: ./nvtest
==28735== NVPROF is profiling process 28735, command: ./nvtest
==28740== Profiling application: ./nvtest
==28736== Profiling application: ./nvtest
==28738== Profiling application: ./nvtest
==28740== Profiling result:
==28738== Profiling result:
==28736== Profiling result:
==28738== Metric result:
Invocations                               Metric Name                        Metric Description         Min         Max         Avg
Device "GeForce GTX 680M (0)"
	Kernel: gpu_kernel(double*)
          1                                  flops_dp                             FLOPS(Double)  ==28740== Metric result:
        16          16          16
Invocations                               Metric Name                        Metric Description         Min         Max         Avg
Device "GeForce GTX 680M (0)"
==28736== Metric result:
Invocations                               Metric Name                        Metric Description         Min         Max         Avg
Device "GeForce GTX 680M (0)"
	Kernel: gpu_kernel(double*)
          1                                  flops_dp                             FLOPS(Double)          16          16          16
	Kernel: gpu_kernel(double*)
          1                                  flops_dp                             FLOPS(Double)          16          16          16
==28735== Profiling application: ./nvtest
==28735== Profiling result:
==28735== Metric result:
Invocations                               Metric Name                        Metric Description         Min         Max         Avg
Device "GeForce GTX 680M (0)"
	Kernel: gpu_kernel(double*)
          1                                  flops_dp                             FLOPS(Double)          16          16          16
```

In case of complex MPI-CUDA program with more processes it would be quite hard to determine which MPI rank specific profiler output belongs to. Output into individual files with `--log-file %p` might be also inconvenient in case of hundreds of processes and still loses information about MPI ranks. In order to resolve this issue, we wrote a simple library that overrides printing functions with buffering. If the library is preloaded with MPI program, the profiler outputs are no longer mixed, and MPI rank is given for each process:

```
$ LD_PRELOAD=./libnvprof.so mpirun -np 4 nvprof --metrics flops_dp ./nvtest 

==30729== NVPROF is profiling process 30729, command: ./nvtest
==30734== NVPROF is profiling process 30734, command: ./nvtest
==30735== NVPROF is profiling process 30735, command: ./nvtest
==30731== NVPROF is profiling process 30731, command: ./nvtest
==30729== ==30734== ==30729== ==30734== ==30735== ==30735== ==30731== ==30731== ==30729== ==30734== ==30735== ==30731== 
Profiling application: ./nvtest
==NVPROF== [RANK:00000] Profiling result:
==NVPROF== [RANK:00000] Metric result:
==NVPROF== [RANK:00000] Invocations                               Metric Name                        Metric Description         Min         Max         Avg
==NVPROF== [RANK:00000] Device "GeForce GTX 680M (0)"
==NVPROF== [RANK:00000] 	Kernel: gpu_kernel(double*)
==NVPROF== [RANK:00000]           1                                  flops_dp                             FLOPS(Double)          16          16          16
==NVPROF== [RANK:00000] 

Profiling application: ./nvtest
==NVPROF== [RANK:00001] Profiling result:
==NVPROF== [RANK:00001] Metric result:
==NVPROF== [RANK:00001] Invocations                               Metric Name                        Metric Description         Min         Max         Avg
==NVPROF== [RANK:00001] Device "GeForce GTX 680M (0)"
==NVPROF== [RANK:00001] 	Kernel: gpu_kernel(double*)
==NVPROF== [RANK:00001]           1                                  flops_dp                             FLOPS(Double)          16          16          16
==NVPROF== [RANK:00001] 

Profiling application: ./nvtest
==NVPROF== [RANK:00002] Profiling result:
==NVPROF== [RANK:00002] Metric result:
==NVPROF== [RANK:00002] Invocations                               Metric Name                        Metric Description         Min         Max         Avg
==NVPROF== [RANK:00002] Device "GeForce GTX 680M (0)"
==NVPROF== [RANK:00002] 	Kernel: gpu_kernel(double*)
==NVPROF== [RANK:00002]           1                                  flops_dp                             FLOPS(Double)          16          16          16
==NVPROF== [RANK:00002] 

Profiling application: ./nvtest
==NVPROF== [RANK:00003] Profiling result:
==NVPROF== [RANK:00003] Metric result:
==NVPROF== [RANK:00003] Invocations                               Metric Name                        Metric Description         Min         Max         Avg
==NVPROF== [RANK:00003] Device "GeForce GTX 680M (0)"
==NVPROF== [RANK:00003] 	Kernel: gpu_kernel(double*)
==NVPROF== [RANK:00003]           1                                  flops_dp                             FLOPS(Double)          16          16          16
==NVPROF== [RANK:00003]
```

Below is the library source code:

```c++
//$ cat nvprof.cpp 

#include <dlfcn.h>
#include <iomanip>
#include <iostream>
#include <malloc.h>
#include <map>
#include <mpi.h>
#include <pthread.h>
#include <sstream>
#include <stdio.h>
#include <stdlib.h>
#include <string>
#include <string.h>

#define MPI_ERR_CHECK(call)                                          \
	do {                                                         \
		int err = call;                                      \
		if (err != MPI_SUCCESS) {                            \
			char errstr[MPI_MAX_ERROR_STRING];           \
			int szerrstr;                                \
			MPI_Error_string(err, errstr, &szerrstr);    \
			fprintf(stderr, "MPI error at %s:%i : %s\n", \
				__FILE__, __LINE__, errstr);         \
			abort();                                     \
		}                                                    \
	}                                                            \
	while (0)

#define libc "libc.so.6"
static void* libc_handle = NULL;

#define libpthread "libpthread.so.0"
static void* libpthread_handle = NULL;

#define bind_lib(lib) \
if (!lib##_handle) \
{ \
	lib##_handle = dlopen(lib, RTLD_NOW | RTLD_GLOBAL); \
	if (!lib##_handle) \
	{ \
		fprintf(stderr, "Error loading %s: %s", lib, dlerror()); \
		abort(); \
	} \
}

#define bind_sym(lib, sym, retty, ...) \
typedef retty (*sym##_func_t)(__VA_ARGS__); \
static sym##_func_t sym##_real = NULL; \
if (!sym##_real) \
{ \
	sym##_real = (sym##_func_t)dlsym(lib##_handle, #sym); \
	if (!sym##_real) \
	{ \
		fprintf(stderr, "Error loading %s : %s", #sym, dlerror()); \
		abort(); \
	} \
}

using namespace std;

struct buffer_t
{
	int on;
	char* content;
	char* content_ptr;
	
	buffer_t() : on(0), content(NULL) { }
};

map<unsigned long long, struct buffer_t> buffers;
pthread_mutex_t buffers_mutex = PTHREAD_MUTEX_INITIALIZER; 

extern "C" void print_buffers(void)
{
	for (map<unsigned long long, struct buffer_t>::iterator i = buffers.begin(),
		e = buffers.end(); i != e; i++)
	{
		// Put process signature at the start of each line.
		MPI_ERR_CHECK( MPI_Init(NULL, NULL) );
		int rank;
		MPI_ERR_CHECK( MPI_Comm_rank(MPI_COMM_WORLD, &rank) );
		MPI_ERR_CHECK( MPI_Finalize() );
		
		stringstream ss;
		ss << "==NVPROF== [RANK:" << setfill('0') << setw(5) << rank << "] ";
		string sign = ss.str();
		string result = i->second.content;
		free(i->second.content);
		for (size_t index = 0; ; )
		{
			index = result.find("\n", index);
			if (index == string::npos) break;
			index++;
			result.insert(index, sign);
			index += sign.length();
		}
		
		printf("\n%s\n", result.c_str());
		fflush(stdout);
	}
}

extern "C" int vfprintf(FILE* stream, const char* format, va_list arg)
{
	bind_lib(libc);
	bind_sym(libc, vfprintf, int, FILE*, const char*, va_list);

	if (!strcmp(format, "Profiling application: %s\n"))
	{
		atexit(print_buffers);
	
		// Start bufferizing profiler output.
		buffer_t buffer;
		buffer.on = 1;
		buffer.content = (char*)malloc(32768);
		buffer.content[0] = '\0';
		buffer.content_ptr = buffer.content;
		pthread_mutex_lock(&buffers_mutex);
		buffers[(unsigned long long)pthread_self()] = buffer;
		pthread_mutex_unlock(&buffers_mutex);
	}
	
	if (buffers[(unsigned long long)pthread_self()].on)
	{
		char** content_ptr = &buffers[(unsigned long long)pthread_self()].content_ptr;
		int length = vsprintf(*content_ptr, format, arg);
		*content_ptr += length;
		return length;
	}
	
	int status = vfprintf_real(stream, format, arg);
	
	return status;
}
```

Build with the following command:

```
$ mpicxx -g -fPIC nvprof.cpp -shared -o libnvprof.so
```
