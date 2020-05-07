---
layout: post
title: "How to find CUDA's version of LLVM backend"
tags:
  - Software Engineering
thumbnail_path: blog/2014-07-14-how-to-find-cuda-s-version-of-llvm-backend/nvcc-llvm-ir.png
---

It is well-known that CUDA toolkit uses LLVM backend, but the used version number is not shown. We can use gdb and LLVM API function to print the version string:

```
$ gdb /opt/cuda/nvvm/bin/cicc
(gdb) start
(gdb) p 'llvm::cl::PrintVersionMessage'()
Low Level Virtual Machine [http://llvm.org/](http://llvm.org/):
  llvm version 3.0
  Optimized build.
  Built Mar 13 2014 (11:31:40).
  Host: i386-pc-linux-gnu
  Host CPU: i686
```

So, CUDA 6.0 uses LLVM 3.0. Current LLVM stable release is 3.4.2.
