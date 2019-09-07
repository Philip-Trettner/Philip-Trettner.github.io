---
layout: post
current: post
cover:  assets/images/tunnel-speed.jpeg
navigation: True
title: "Performance of std::function"
date: 2019-09-07 01:00:00
tags: [c++, optimization]
class: post-template
subclass: 'post'
author: philip
excerpt: 'How bad is it really?'
---

Everyone knows that you have to avoid `std::function` if you care about performance.

But is it really true?
How bad is it?

## Nanobenchmarking `std::function`

Benchmarking is hard.
Microbenchmarking is a dark art.
Many people insist that nanobenchmarking is out of the reach for us mortals.

But that won't stop us:
let's benchmark the overhead of creating and calling a `std::function`.

We have to tread extra carefully here.
Modern desktop CPUs are insanely complex, often with deep pipelines, out-of-order execution, sophisticated branch prediction, prefetching, multiple level of caches, hyperthreading, and many more arcane performance-enhancing features.

The other enemy is the compiler.

> Any sufficiently advanced optimizing compiler is indistinguishable from magic.

We'll have to make sure that our code-to-be-benchmarked is not being optimized away.
Luckily, `volatile` is still not fully deprecated and can be (ab)used to prevent many optimizations.
In this post we will only measure throughput (how long does it take to call the same function 1000000 times?).
We're going to use the following scaffold:

```cpp
template<class F>
void benchmark(F&& f, float a_in = 0.0f, float b_in = 0.0f)
{
    auto constexpr count = 1'000'000;

    volatile float a = a_in;
    volatile float b = b_in;
    volatile float r;

    auto const t_start = std::chrono::high_resolution_clock::now();
    for (auto i = 0; i < count; ++i)
        r = f(a, b);
    auto const t_end = std::chrono::high_resolution_clock::now();

    auto const dt = std::chrono::duration<double>(t_end - t_start).count();
    std::cout << dt / count * 1e9 << " ns / op" << std::endl;
}
```

Double checking with [godbolt](https://godbolt.org/z/fjBN1a) we can verify that the compiler is not optimizing the function body even though we only compute `0.0f + 0.0f` in a loop.
The loop itself has some overhead and sometimes the compiler will unroll parts of the loop.

## Baseline

Our subject in the following benchmarks with be an Intel Core i9-9900K running at 4.8 GHz (a modern high-end consumer CPU at the time of writing).
The code is compiled with `clang-7` and the `libcstd++` standard library using `-O2` and `-march=native`.

We start with a few basic tests:

```cpp
benchmark([](float, float) { return 0.0f; });      // 0.21 ns / op (1 cycle / op)
benchmark([](float a, float b) { return a + b; }); // 0.22 ns / op (1 cycle / op)
benchmark([](float a, float b) { return a / b; }); // 0.62 ns / op (3 cycles / op)
```

The baseline is about 1 cycle per operation and the `a / b` test verifies that we can reproduce the throughput of basic operations (a good reference is [AsmGrid](https://asmjit.com/asmgrid/), X86 Perf on the upper right).
(I've repeated all benchmarks multiple times and chose the mode of the distribution.)


## Calling Functions

The first thing we want to know: How expensive is a function call?

```cpp
using fun_t = float(float, float);

// inlineable direct call
float funA(float a, float b) { return a + b; }

// non-inlined direct call
__attribute__((noinline)) float funB(float a, float b) { return a + b; }

// non-inlined indirect call
fun_t* funC; // set externally to funA

// visible lambda
auto funD = [](float a, float b) { return a + b; };

// std::function with visible function
auto funE = std::function<fun_t>(funA);

// std::function with non-inlined function
auto funF = std::function<fun_t>(funB);

// std::function with function pointer
auto funG = std::function<fun_t>(funC);

// std::function with visible lambda
auto funH = std::function<fun_t>(funD);

// std::function with direct lambda
auto funI = std::function<fun_t>([](float a, float b) { return a + b; });
```

The results:

```cpp
benchmark(funA); // 0.22 ns / op (1 cycle / op)
benchmark(funB); // 1.04 ns / op (5 cycles / op)
benchmark(funC); // 1.04 ns / op (5 cycles / op)
benchmark(funD); // 0.22 ns / op (1 cycle / op)
benchmark(funE); // 1.67 ns / op (8 cycles / op)
benchmark(funF); // 1.67 ns / op (8 cycles / op)
benchmark(funG); // 1.67 ns / op (8 cycles / op)
benchmark(funH); // 1.25 ns / op (6 cycles / op)
benchmark(funI); // 1.25 ns / op (6 cycles / op)
```

## Constructing a `std::function`

We can also measure how long it takes to construct or copy a `std::function`:

```cpp
std::function<float(float, float)> f;

benchmark([&]{ f = {}; });   // 0.42 ns / op ( 2 cycles / op)
benchmark([&]{ f = funA; }); // 4.37 ns / op (21 cycles / op)
benchmark([&]{ f = funB; }); // 4.37 ns / op (21 cycles / op)
benchmark([&]{ f = funC; }); // 4.37 ns / op (21 cycles / op)
benchmark([&]{ f = funD; }); // 1.46 ns / op ( 7 cycles / op)
benchmark([&]{ f = funE; }); // 5.00 ns / op (24 cycles / op)
benchmark([&]{ f = funF; }); // 5.00 ns / op (24 cycles / op)
benchmark([&]{ f = funG; }); // 5.00 ns / op (24 cycles / op)
benchmark([&]{ f = funH; }); // 4.37 ns / op (21 cycles / op)
benchmark([&]{ f = funI; }); // 4.37 ns / op (21 cycles / op)
```

The result of `f = funD` suggests that constructing a `std::function` directly from a lambda is pretty fast.
Let's check that when using different capture sizes:

```cpp
struct b4 { int32_t x; };
struct b8 { int64_t x; };
struct b16 { int64_t x, y; };

benchmark([&]{ f = [](float, float) { return 0; }; });          // 1.46 ns / op ( 7 cycles / op)
benchmark([&]{ f = [x = b4{}](float, float) { return 0; }; });  // 4.37 ns / op (21 cycles / op)
benchmark([&]{ f = [x = b8{}](float, float) { return 0; }; });  // 4.37 ns / op (21 cycles / op)
benchmark([&]{ f = [x = b16{}](float, float) { return 0; }; }); // 1.66 ns / op ( 8 cycles / op)
```

I didn't have the patience to untangle the assembly or the `libcstd++` implementation to check where this behavior originates.
You obviously have to pay for the capture and I think what we see here is a strange interaction between some kind of small function optimization and the compiler hoisting the construction of `b16{}` out of our measurement loop.


## Summary

I think there is a lot of fearmongering regarding `std::function`, not all of it is justified.

My benchmarks suggest that on a modern microarchitecture the following overhead can be expected on hot data and instruction caches:

| calling a non-inlined function | 4 cycles |
| calling a function pointer | 4 cycles |
| calling a `std::function` of a lambda | 5 cycles |
| calling a `std::function` of a function or function pointer | 7 cycles |
| constructing an empty `std::function` | 7 cycles |
| constructing a `std::function` from a function or function pointer | 21 cycles |
| copying a `std::function` | 21..24 cycles |
| constructing a `std::function` from a non-capturing lambda | 7 cycles |
| constructing a `std::function` from a capturing lambda | 21+ cycles |

A word of caution: the benchmarks really only represent the overhead relative to `a + b`.
Different function show slightly different overhead behavior as they might use different scheduler ports and execution units that might overlap differently with what the loop requires.
Also, a lot of this depends on how willing the compiler is to inline.

We've only measured the throughput.
The results are only valid for "calling the same function many times with different arguments", not for "calling many different functions".
But that is a topic for another post.
