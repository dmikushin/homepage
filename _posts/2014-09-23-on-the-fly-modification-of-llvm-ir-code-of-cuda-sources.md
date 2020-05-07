---
layout: post
title: "On-the-fly modification of LLVM IR code of CUDA sources"
tags:
  - Software Engineering
thumbnail_path: blog/2014-09-23-on-the-fly-modification-of-llvm-ir-code-of-cuda-sources/nvcc-llvm-ir.png
---

Largely thanks to [LLVM](llvm.org), in recent years we've seen a significant increase of interest to domain-specific compilation tools research & development. With the release of PTX backends by NVIDIA (opensource [NVPTX](http://llvm.org/docs/NVPTXUsage.html) and proprietary [libNVVM](https://developer.nvidia.com/cuda-llvm-compiler)), construction of custom LLVM-driven compilers for generating GPU binaries also becomes possible. However, two questions are still remaining:

![alt text](\assets\img\blog\2014-09-23-on-the-fly-modification-of-llvm-ir-code-of-cuda-sources\nvcc-llvm-ir.png "Logo Title Text 1")

How to customize the CUDA source compilation?
What is the NVIDIA's best set of GPU-specific LLVM optimizations and how to continue modifying IR after applying them?
In order to answer these two questions, we have created a special dynamic library. Being attached to NVIDIA CUDA compiler, this library exposes unoptimized and optimized LLVM IR code to the user and allows its on-the-fly modification. As result, domain-specific compiler developer receives flexibility e.g. to re-target CUDA-generated LLVM IR to different architectures, or to make additional modifications to IR after executing NVIDIA's optimizations.

Source code, sample modification and description are available on our [GitHub page](https://github.com/apc-llc/nvcc-llvm-ir/tree/public). Tested with CUDA 6.0.
