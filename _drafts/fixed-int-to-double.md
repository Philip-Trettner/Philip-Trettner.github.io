---
layout: post
current: post
cover:  assets/images/floating-island.jpg
navigation: True
title: "Converting Wide Integers to Double"
date: 2021-06-20 02:00:00
tags: [c++, optimization]
class: post-template
subclass: 'post'
author: philip
excerpt: "Efficient i128, i192, i256 to double via this one weird trick"
---

TL;DR: In this article, we will develop a highly efficient conversion of wide integers (e.g. 192 bit or 256 bit) to floating point (in our case `double`).
Main takeaways:

* `u64 -> double` is a lot slower than `i64 -> double` on current CPUs
* two's complement handling can be unified via negative weight for MSB
* fastest conversion uses a 2-entry lookup table

# Introduction

In most cases, 64 bit integers provide enough range for all your needs.
But occasionally, larger integers are needed.
If you need arbitrarily large numbers, libraries like [gmp](https://gmplib.org/) are your best bet.
While highly optimized, they still tend to be comparatively slow.
Arbitrary-precision math requires loops and potentially allocations for basically every operation.

However, sometimes the numbers are not actually arbitrarily large, just too large for 64 bit.
A common use case is asymmetric cryptography, where large, but fixed-size numbers are multiplied.

My main field of study is high-performance geometry processing.
One of the subfields that I research is mesh Booleans and constructive solid geometry (CSG).
Basically, computing unions and intersections of meshes.
If you try to solve this using floating points numbers, you end up in precision pandemonium.
All really robust approaches use exact arithmetic, for example using gmp rationals or exact floating point predicates.

We published a paper about [Fast Exact Booleans for Iterated CSG using Octree-Embedded BSPs](http://www.graphics.rwth-aachen.de/publication/03331/) that uses a new approach:
All vertex positions are actually wide integers, for example four 192 bit values (in homogeneous coordinates).
This enabled us to compute everything 100% exact but without the typical performance penalty.

One of the intermediate steps involved converting the integer coordinates back to floating point.
The rest of this article will focus on how to get the wide integer to double conversion as efficient as possible.

# Baseline Version

Our running example will be a 192 bit signed integer, interpreted in two's complement.
Internally, we store it as 3 "bags" of 64 bit:

```cpp
struct i192
{
    uint64_t w0; // least significant word
    uint64_t w1;
    uint64_t w2; // most significant word
};
```

If we have unary negation, there is a simple way to implement the conversion to double:

```cpp
double to_double(i192 i)
{
    if (i < 0) 
        return -to_double(-i);

    return i.w0 + i.w1 * 0x1p64 + i.w2 * 0x1p128;
}
```

We basically reduce the signed case to the unsigned case, convert each 64 bit word to double and sum them up with appropriate weights.
`0x1p64` and `0x1p128` are fancy [hexadecimal floating point literals](https://en.cppreference.com/w/cpp/language/floating_literal) that are available since C++17.
They are a simple and efficient way to say \\(2^{64}\\) and \\(2^{128}\\) in `double`.
Thus, we really just compute
\\[ w_0 + 2^{64} w_1 + 2^{128} w_2. \\]

Even if we ignore the sign check, there is something fishy with the [resulting assembly](https://godbolt.org/z/Gd1P6eMK8):
```cpp
vmovsd xmm0, qword ptr [rsp + 8]
vunpcklps xmm0, xmm0, xmmword ptr [rip + .LCPI0_0]
vsubpd xmm0, xmm0, xmmword ptr [rip + .LCPI0_1]
vpermilpd xmm1, xmm0, 1
vmovdqu xmm2, xmmword ptr [rsp + 16]
vpxor xmm3, xmm3, xmm3
vpblendd xmm3, xmm2, xmm3, 10
vpor xmm3, xmm3, xmmword ptr [rip + .LCPI0_2]
vaddsd xmm0, xmm1, xmm0
vpsrlq xmm1, xmm2, 32
vpor xmm1, xmm1, xmmword ptr [rip + .LCPI0_3]
vsubpd xmm1, xmm1, xmmword ptr [rip + .LCPI0_4]
vaddpd xmm1, xmm3, xmm1
vmulpd xmm1, xmm1, xmmword ptr [rip + .LCPI0_5]
vaddsd xmm0, xmm1, xmm0
vpermilpd xmm1, xmm1, 1 # xmm1 = xmm1[1,0]
vaddsd xmm0, xmm0, xmm1
```
and don't get me started on the gcc version, which is littered with branches.

# `u64 -> double` vs `i64 -> double`

It turns out that converting `uint64_t` to `double` is actually more expensive than I thought.
Especially because `int64_t` to `double` is extremely cheap ([godbolt](https://godbolt.org/z/YMn5TjcPf)):
```cpp
to_double(uint64_t): // clang 11
  vmovq xmm0, rdi
  vpunpckldq xmm0, xmm0, xmmword ptr [rip + .LCPI0_0]
  vsubpd xmm0, xmm0, xmmword ptr [rip + .LCPI0_1]
  vpermilpd xmm1, xmm0, 1
  vaddsd xmm0, xmm1, xmm0
  ret

to_double(uint64_t): // gcc 9
  vxorps xmm0, xmm0, xmm0
  test rdi, rdi
  js .L2
  vcvtsi2sd xmm0, xmm0, rdi
  ret
.L2:
  mov rax, rdi
  shr rax
  and edi, 1
  or rax, rdi
  vcvtsi2sd xmm0, xmm0, rax
  vaddsd xmm0, xmm0, xmm0
  ret

to_double(int64_t):
  vcvtsi2sd xmm0, xmm0, rdi
  ret
```
How fast is `int64_t` to `double`?
According to [AsmGrid](https://asmjit.com/asmgrid/) (X64Perf -> `vcvtsi2sd`), we have _a single cycle_ latency and throughput, which is extremely fast.

Fun fact: There is a fast `uint64_t` to `double` instruction, called [vcvtusi2sd](https://www.felixcloutier.com/x86/vcvtusi2sd), but it requires AVX512F.
With Skylake-X we have a few desktop processors that support this, but it's far from widely available.

# Unified Two's Complement Handling

TODO: u64 slow, choose i64

TODO: negative weight of MSB

# 2-Entry LUT

TODO: branching still sucky

TODO: use 1bit LUT

# Benchmarks

TODO: quick-bench? https://quick-bench.com/q/PCYbRfhdMrmvzYJjhdiAvH9rSuo

# Conclusion

TODO

(_Title image from [pixabay](https://pixabay.com/illustrations/home-mountains-fantasy-floating-5889366/)_)
