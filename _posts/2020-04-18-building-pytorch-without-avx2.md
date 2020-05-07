---
layout: post
title: "Building PyTorch without AVX2 on MacOS"
tags:
- Building
- Software Engineering
- PyTorch
- AVX2
thumbnail_path: blog/2020-04-18-building-pytorch-without-avx2/pytorch.png
pytorch_url: "https://github.com/pytorch/pytorch"
pytorch_fork_url: "https://github.com/dmikushin/pytorch-no-avx2-macos"
---

In order to quickly explore [PyTorch]({{ page.pytorch_url }}) internals, I decided to compile and install a Debug build on my local machine. The first problem was that modern Clang surprisingly crashes on compiling Sobol RNG initial state setup, which is a very regular piece of code:

```c++
/// This is a core function to initialize the main state variable of a `SobolEngine`.
/// `dimension` is passed explicitly as well (see why above)
Tensor& _sobol_engine_initialize_state_(Tensor& sobolstate, int64_t dimension) {
  TORCH_CHECK(sobolstate.dtype() == at::kLong,
           "sobolstate needs to be of type ", at::kLong);

  /// First row of `sobolstate` is 1
  sobolstate.select(0, 0).fill_(1);

  /// Use a tensor accessor for `sobolstate`
  auto ss_a = sobolstate.accessor<int64_t, 2>();
  for (int64_t d = 0; d < dimension; ++d) {
    int64_t p = poly[d];
    int64_t m = bit_length(p) - 1;

    for (int64_t i = 0; i < m; ++i) {
      ss_a[d][i] = initsobolstate[d][i];
    }

    for (int64_t j = m; j < MAXBIT; ++j) {
      int64_t newv = ss_a[d][j - m];
      int64_t pow2 = 1;
      for (int64_t k = 0; k < m; ++k) {
        pow2 <<= 1;
        if ((p >> (m - 1 - k)) & 1) {
          newv = newv ^ (pow2 * ss_a[d][j - k - 1]);
        }
      }
      ss_a[d][j] = newv;
    }
  }

  Tensor pow2s = at::pow(2, at::native::arange((MAXBIT - 1), -1, -1, sobolstate.options()));
  sobolstate.mul_(pow2s);
  return sobolstate;
}
```

Turns out, the crash is likely connected to the fact that `initsobolstate` is a very large static array. A patch that worked (with some additional improvements) basically hides array nature behind a `memcpy` call:

```patch
--- a/aten/src/ATen/native/SobolEngineOps.cpp
+++ b/aten/src/ATen/native/SobolEngineOps.cpp
@@ -129,21 +129,19 @@ Tensor& _sobol_engine_initialize_state_(Tensor& sobolstate, int64_t dimension) {
 
   /// Use a tensor accessor for `sobolstate`
   auto ss_a = sobolstate.accessor<int64_t, 2>();
-  for (int64_t d = 0; d < dimension; ++d) {
+  for (int32_t d = 0; d < dimension; ++d) {
     int64_t p = poly[d];
-    int64_t m = bit_length(p) - 1;
+    int32_t m = bit_length(p) - 1;
 
-    for (int64_t i = 0; i < m; ++i) {
-      ss_a[d][i] = initsobolstate[d][i];
-    }
+    memcpy(&ss_a[d][0], &initsobolstate[d][0], m * sizeof(ss_a[d][0]));
 
-    for (int64_t j = m; j < MAXBIT; ++j) {
+    for (int32_t j = m; j < MAXBIT; ++j) {
       int64_t newv = ss_a[d][j - m];
-      int64_t pow2 = 1;
-      for (int64_t k = 0; k < m; ++k) {
+      int64_t pow2 = 1LL;
+      for (int32_t k = 0; k < m; ++k) {
         pow2 <<= 1;
-        if ((p >> (m - 1 - k)) & 1) {
-          newv = newv ^ (pow2 * ss_a[d][j - k - 1]);
+        if ((p >> (m - 1 - k)) & 1LL) {
+          newv ^= pow2 * ss_a[d][j - k - 1];
         }
       }
       ss_a[d][j] = newv;
```

Another problem is vectorization support. By default, PyTorch enables all vectorization options (such as AVX, AVX2, AVX512 on x86) that are supported by compiler. Although vectorization flags are enabled only for a small subset of source files, such policy leaves dated hardware in a complicated situation. In order to disable everything except AVX for 2012' Macbook Pro, I have to come up with a set of CMake options:

```
mkdir build
cd build 
cmake -DCMAKE_BUILD_TYPE=Debug -DPYTHON_EXECUTABLE=/usr/local/bin/python3 -DDISABLE_AVX2:BOOL=TRUE -DCXX_AVX2_FOUND:BOOL=FALSE -DC_AVX2_FOUND:BOOL=FALSE -DDISABLE_AVX512F:BOOL=TRUE ..
```

Once CMake is configured, get back to the root folder and continue with the actual building with setup script, as long as we need Python wheel:

```
cd ..
python3 setup.py bdist_wheel
```

Lastly, the wheel may place native libraries to an unexpected location. In this case you will be notified by a runtime error. Moving libraries to `/usr/local/lib/python3.7/site-packages/torch/lib` should resolve the issue.

For the reference, here is a [PyTorch fork]({{ page.pytorch_fork_url }}) that could be used to reproduce my experience.

