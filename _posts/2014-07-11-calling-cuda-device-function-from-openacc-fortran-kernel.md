---
layout: post
title: "Calling CUDA device function from OpenACC Fortran kernel"
tags:
  - Software Engineering
thumbnail_path: blog/2014-07-11-calling-cuda-device-function-from-openacc-fortran-kernel/openacc.jpg
---

OpenACC is known to be a fast method of developing quite efficient GPU-enabled applications. It is also possible to mix CUDA kernels and libraries with OpenACC kernels in single source. But do you know it is also possible to call CUDA device functions from within OpenACC kernels? This feature enables, for instance, the use of libraries with device functions (e.g. CURAND) and dynamic parallelism in OpenACC. We've developed an example program to show you how CUDA device subroutine could be called from OpenACC kernel in Fortran:

![alt text](\assets\img\blog\2014-07-11-calling-cuda-device-function-from-openacc-fortran-kernel\openacc.jpg "Logo Title Text 1")

```c++
program main
use openacc
use device_subroutine_module

!$acc routine (device_subroutine) seq
integer, dimension(:), allocatable :: a
integer, dimension(:), allocatable :: b
integer :: i

allocate(a(10), b(10))

!$acc kernels copyout (a,b)
!$acc loop
do i = 1, 10
    call device_subroutine(i, a, b)
enddo

!$acc end kernels
print *, a

deallocate(a, b)

end program
```

Device subroutine could be implemented in CUDA Fortran or CUDA C. In latter case we need Fortran interface:

```c++
module device_subroutine_module
use cudafor
interface
    attributes(device) subroutine device_subroutine(i, a, b)
    INTEGER, value :: i
    integer :: a(*), b(*)
    end subroutine
end interface
end module
```

Fortran interface above connects our Fortran application with CUDA C device function implementation:

```c++
extern "C" __device__ void my_subroutine_(int i, int* a, int* b)
{
    a[i - 1] = i;
    b[i - 1] = i + 10;
}
```

Compile and link everything together with pgfortran and nvcc:

```
COMP=pgfortran
PGI_ACC_FLAGS=-acc -Minfo=accel -ta=nvidia:cuda6.0,cc30

all: program

called_function_interface.o: called_function.f90
    $(COMP) -Mcuda=rdc -c $< -o $@ $(LIB_PGI)

called_function.o: called_function.cu
    nvcc -rdc=true -gencode arch=compute_30,code=compute_30 -c $< -o $@

libcalled_function.a: called_function.o called_function_interface.o
    ar r libcalled_function.a $^

caller_function.o: caller_function.f90 called_function_interface.o
    $(COMP) $(PGI_ACC_FLAGS) -c $< -o caller_function.o  $(LIB_PGI)

program: caller_function.o libcalled_function.a
    $(COMP) $(PGI_ACC_FLAGS) $< -o program -L. -lcalled_function -lcudafor -L$(shell dirname $(shell which nvcc))/../lib64 -lcudart

clean:
    rm -rf *.o *.mod libcalled_function.a program
```

Our test program sucessfully fills arrays:

```
$ ./program
            1            2            3            4            5            6
            7            8            9           10
```
