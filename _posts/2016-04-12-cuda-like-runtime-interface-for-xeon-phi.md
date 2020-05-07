---
layout: post
title: "CUDA-like runtime interface for Xeon Phi"
tags:
  - Software Engineering
thumbnail_path: blog/2016-04-12-cuda-like-runtime-interface-for-xeon-phi/micrt.png
---

The performance power of GPUs could be exposed to applications using two principal kinds of programming interfaces: with manual parallel programming (CUDA or OpenCL), or with directive-based extensions relying on compiler's capabilities of semi-automatic parallelization (OpenACC and OpenMP4). Unlike for GPUs, Intel has never offered an explicit CUDA-like interface for their Xeon Phi accelerators to general public, leaving OpenMP offloading directives as the only programming option.

![alt text](\assets\img\blog\2016-04-12-cuda-like-runtime-interface-for-xeon-phi\micrt.png "Logo Title Text 1")

Based on liboffloadmic, we have prototyped "micrt" - a programming interface to execute memory transfers and kernels, similarly to CUDA runtime. Find the code example and building instructions [here](https://github.com/apc-llc/liboffloadmic).
