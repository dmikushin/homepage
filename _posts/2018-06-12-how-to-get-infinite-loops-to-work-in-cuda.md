---
layout: post
title: "How to get infinite loops to work in CUDA"
tags:
  - CUDA
  - Software Engineering
thumbnail_path: blog/2018-06-12-how-to-get-infinite-loops-to-work-in-cuda/infinite-loop.png
---

The CUDA compiler does not handle infinite loops properly. For instance, the loop below will be completely eliminated from the resulting assembly, along with its contents. This situation seems to be [known](https://stackoverflow.com/questions/10436228/cuda-infinite-kernel) at least since 2012.

```c++
while(1)
{
  ...
}
```

Although this behavior is valid according to the standard, there are certain situations where infinite loop on GPU is required. For instance, ine may need to run a background service with configuration updates periodically sent from host over host-mapped memory.

One solution to enforce infinite loop is:

```c++
volatile int infinity = 1;
while (infinity)
{
  ...
}
```
