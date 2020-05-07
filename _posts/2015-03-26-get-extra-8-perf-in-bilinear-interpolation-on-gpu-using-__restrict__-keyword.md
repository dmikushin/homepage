---
layout: post
title: "Get extra 8% perf in bilinear interpolation on GPU using __restrict__ keyword"
tags:
  - Software Engineering
thumbnail_path: blog/2015-03-26-get-extra-8-perf-in-bilinear-interpolation-on-gpu-using-__restrict__-keyword/GPU-Pro-Tip.png
---

Starting from GK110 (Tesla Kepler), "const **restrict**" annotation on kernel argument has an extra GPU-specific meaning: accesses to that argument should go through the texture cache. As an example, we use GPU bilinear interpolation, which is a compute-bound problem. Replacing of "RGBApixel* pixels" by "const RGBApixel* **restrict** pixels" in **global** and **device** functions instructs the compiler to emit LDG instructions for pixels loading instead of generic LD:

![alt text](\assets\img\blog\2015-03-26-get-extra-8-perf-in-bilinear-interpolation-on-gpu-using-__restrict__-keyword\GPU-Pro-Tip.png "Logo Title Text 1")

```
$ cuobjdump -sass no_restrict | grep LD
        /*01d8*/                   LD.E R13, [R8+0x4];
        /*0210*/                   LD.E R14, [R8];
        /*0248*/                   LD.E R8, [R4+0x4];
        /*0290*/                   LD.E R11, [R4];
$ cuobjdump -sass restrict | grep LD
        /*01a0*/                   LDG.E R7, [R16];
        /*01b0*/                   LDG.E R9, [R8];
        /*01e0*/                   LDG.E R8, [R16];
        /*01f0*/                   LDG.E R5, [R14];
```

If **restrict** pointer does not produce LDG, then the keyword is not applicable or is used incorrectly.

Code version with "const **restrict**" demonstrates extra speedup of 8%:

```
$ ./no_restrict hst_lagoon_detail.bmp
Image: BMP 4778 x 4856
GPU kernel time = 0.013719 sec
$ ./restrict hst_lagoon_detail.bmp
Image: BMP 4778 x 4856
GPU kernel time = 0.012686 sec
```

Download our demo code [here](https://parallel-computing.pro/articles/restrict/restrict.zip).
