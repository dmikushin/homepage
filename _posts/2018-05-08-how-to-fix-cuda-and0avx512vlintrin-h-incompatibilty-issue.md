---
layout: post
title: "How to fix CUDA and avx512vlintrin.h incompatibilty issue"
tags:
- CUDA
- Software Engineering
- GCC
thumbnail_path: blog/2018-05-08-how-to-fix-cuda-and0avx512vlintrin-h-incompatibilty-issue/gcc.png
---

Recent 5.x and 6.x GCC compilers are causing NVCC to produce the following kind of weird compile errors:

```
/usr/lib/gcc/x86_64-linux-gnu/5/include/avx512vlintrin.h(10919): error: argument of type "const void *" is incompatible with parameter of type "const long long *"
```

In order to track down the issue, pick up the failing NVCC command line and add -M flag (or replace -c flag with -M flag) to print down the structure of used header files:

```
$ /opt/cuda/bin/nvcc ... -M -o outfile
```

While walking through the output, notice, which system header sits on top of problematic avx512... header. In our case, it was <algorithm>:

```
$ cat outfile
...
    /usr/include/c++/5/algorithm \
    /usr/include/c++/5/bits/stl_algo.h \
    /usr/include/c++/5/bits/algorithmfwd.h \
    /usr/include/c++/5/bits/stl_heap.h \
    /usr/include/c++/5/random \
    /usr/include/c++/5/limits \
    /usr/include/c++/5/bits/random.h \
    /usr/include/c++/5/bits/uniform_int_dist.h \
    /usr/include/x86_64-linux-gnu/c++/5/bits/opt_random.h \
    /usr/lib/gcc/x86_64-linux-gnu/5/include/x86intrin.h \
    /usr/lib/gcc/x86_64-linux-gnu/5/include/ia32intrin.h \
    /usr/lib/gcc/x86_64-linux-gnu/5/include/pmmintrin.h \
    /usr/lib/gcc/x86_64-linux-gnu/5/include/tmmintrin.h \
    /usr/lib/gcc/x86_64-linux-gnu/5/include/ammintrin.h \
    /usr/lib/gcc/x86_64-linux-gnu/5/include/smmintrin.h \
    /usr/lib/gcc/x86_64-linux-gnu/5/include/popcntintrin.h \
    /usr/lib/gcc/x86_64-linux-gnu/5/include/wmmintrin.h \
    /usr/lib/gcc/x86_64-linux-gnu/5/include/immintrin.h \
    /usr/lib/gcc/x86_64-linux-gnu/5/include/avxintrin.h \
...
```

After removing `#include <algorithm>`, the compiler error is gone. If <algorithm> could not be removed right away due to complex code dependencies, you can workaround by splitting your source file into CUDA source and <algorithm>-dependent C++ source to be compiled solely by GCC.
