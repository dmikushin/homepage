---
layout: post
title: "Using C-Reduce to understand Clang compiler bugs"
tags:
- LLVM
- Software Engineering
thumbnail_path: blog/2022-06-28-using-c-reduce-to-understand-clang-compiler-bugs/llvm.png
---

Suppose we have a crash while compiling huge application from source, e.g. a Python package with native C++ code. A source file fails to compile with the following message:

```
PLEASE submit a bug report to https://bugs.llvm.org/ and include the crash backtrace, preprocessed source, and associated run script.
Stack dump:
0.	Program arguments: /opt/llvm/bin/clang-14 ...
...
Segmentation fault (core dumped)
```

Analyzing source files with many include files is often not feasible. Instead, we can deploy automatic test case reduction tools such as LLVM bugpoint or [C-Reduce](https://github.com/csmith-project/creduce). C-Reduce is actually generic, and can evaluate bugs of any compiler. But in this usecase we focus particulary on Clang.

## Preparation

First preparatory step for C-Reduce is to gather all source code into a single file, which means to perform the preprocessing step. This is required for C-Reduce to work with just a single file, and be able to modify it. Preprocessing could be performed just by adding `-E` in the end of the "Program arguments" command of the crash report above. We will rename the output file to `bug.cpp`.

## Building C-Reduce

Assuming that we may need to work inside CI environment, such as Manylinux CentOS Docker container, we will use yum and compile C-Reduce from source:

```
git clone https://github.com/csmith-project/creduce.git
cd creduce/
```

* Each major verson of LLVM introduced breaking changes, so maintainers provide branches for each LLVM version.
* Our LLVM is already version 14, but it is still compatible with version 13

```
git checkout llvm-13.0
```

Install prerequisites:

```
yum install ncurses-devel flex perl-Exporter-Lite cpan
cpan -i Getopt::Tabular Regexp::Common File::Which
```

Perform the build, using LLVM toolchain provided in the user-specified folder:

```
PATH=/opt/llvm/bin:$PATH ./configure --prefix=$(pwd)/install
make -j8
make install
```

## Usage

Prepare `bug.sh` evaluation script for C-Reduce to work on:

```
#!/bin/sh
/opt/llvm/bin/clang-14 -cc1 ... -O3 bug.cpp && \
! /opt/llvm/bin/clang-14 -cc1 ... -O0 bug.cpp
```

In our case, build succeeds with `-O3` and fails with `-O0`, so we combine these two outcomes into a logical expression and feed it into C-Reduce, also providing the source file name for reducing:

```
LD_LIBRARY_PATH=/opt/llvm/lib:$LD_LIBRARY_PATH creduce/install/bin/creduce bug.sh bug.cpp
```

`LD_LIBRARY_PATH` should additionally provide the location of LLVM libs.

## Result

```
# creduce/install/bin/creduce bug.sh bug.cpp 
===< 41280 >===
running 3 interestingness tests in parallel
===< pass_unifdef :: 0 >===
===< pass_comments :: 0 >===
===< pass_ifs :: 0 >===
===< pass_includes :: 0 >===
===< pass_line_markers :: 0 >===
(3.5 %, 5281788 bytes)
(5.4 %, 5178731 bytes)
...
===< pass_indent :: final >===
(100.0 %, 256 bytes)
===================== done ====================
```

After several hours of work, the original ~100K lines of code have been automatically reduced to just a few lines, for example just into this small test case that crashes the compiler:

```
extern "C" __attribute__((device)) void __ockl_atomic_add_noret_f32(float *,
                                                                    float);
__attribute__((device)) void a(float *address, float b) {
  __ockl_atomic_add_noret_f32(address, b);
}
```

## References

1. [Using C-Reduce](http://embed.cs.utah.edu/creduce/using/)

