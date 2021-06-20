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

TODO: 192 bit struct via 3x u64, interpret as two's complement

TODO: 3x u64 -> double with different factors (and a negative check with inversion before)

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
