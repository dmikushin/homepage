---
layout: post
title: "One non-obvious reason of 'Illegal instruction' in GPU code"
tags:
- Software Engineering
thumbnail_path: blog/2014-04-12-one-non-obvious-reason-of-illegal-instruction-in-gpu-code/illegal-instruction.png
---

If cuda-gdb throws Program received signal CUDA_EXCEPTION_4, Warp Illegal Instruction. for the following code line:

```asm
[Switching focus to CUDA kernel 295, grid 148, block (0,2,3), thread (0,0,0), device 0, sm 0, warp 26, lane 0]
0x00000000010b39f0 in cos ()
(cuda-gdb) disass
Dump of assembler code for function cos:
...
   0x00000000010b39e8 <+376>:	LDC.64 R32, c[0x3][R12]
(note debugger always points to the next address after problematic instruction, i.e. 0xe8 + 0x8 = 0xf0 in this case)
```

then this means used register index is outside of the legal bounds set by kernel's register count.
