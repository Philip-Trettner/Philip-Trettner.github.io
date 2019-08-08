---
layout: post
current: post
cover:  assets/images/hello-blog.jpg
navigation: True
title: Basic Floating Point Optimizations
date: 2019-08-02 01:00:00
tags: [c++, optimization]
class: post-template
subclass: 'post'
author: philip
excerpt: 'Why is "f + 0.0" slower than "f - 0.0"?'
---

Many posts have been written about elaborate magic involving IEEE 754 floating point numbers.
Some of my favorites include the [fast inverse square root](https://en.wikipedia.org/wiki/Fast_inverse_square_root#History_and_investigation) (commonly attributed to John Carmack though the method is much older), [basically everything from the Random ASCII blog](https://randomascii.wordpress.com/category/floating-point/), and the [reverse-Z depth test](https://developer.nvidia.com/content/depth-precision-visualized) for rendering.

In this post I want to take a look at less flashy, more foundational things.
We're going over a couple small optimizations that we probably expected from our compiler but that are not actually legal or only work due to slightly arcane rules.

While this might apply to other languages as well, I'm most familiar with C++ which I'll be using for this post.
Because C++ is actually a standard (with a big chunk of it dedicated to specifying corner cases), we can actually get authoritative answers for most of our questions!

We will furthermore assume that our C++ implementation uses:

* Two's complement for integers
* IEEE 754 for floating points

(This is currently implementation-defined behavior, but at least [the first might change](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0907r1.html))

As always, it is highly instructive to look at the generated assembly for which we will use the excellent page by [Matt Godbolt](https://godbolt.org/).


## The Curios Case of `f + 0.0`

Most programmers are taught that floating point math is [not associative](https://en.wikipedia.org/wiki/Floating-point_arithmetic#Accuracy_problems) ([for example](https://godbolt.org/z/4MPMs-) `-1e9f + 1e9f + 1 == 1` vs. `-1e9f + (1e9f + 1) == 0`).
The same goes for integers if overflows might be involved.

Sometimes one might come across trivial computations like `i + 0`, `f * 1.0`, `i * 1`, etc..
We might want to spell them out for clarity or consistency or because it's part of a generic, templated algorithm that happens to be instantiated for these values.
It is very tempting to think "my compiler is smart and will do the-right-thing (tm)".

_Does it though?_

As any good lawyer will tell you: it depends.

Integers are easier to reason about (for humans and computers alike) and have less surprises than floats, so `i + 0`, `i - 0`, `i * 1`, and `i / 1` will all be optimized away. (TODO: check + godbolt link)
Sometimes, the compiler can do a strength reduction (TODO: link) and for example replace `i * 2` with `i << 1`, which is semantically identical but faster.
You might think that `i * -1` can be replaced with `-i` (TODO: check + continue)

Nothing is ever easy in floating points.
The typical gang that will ruin your special-case-free reasoning is `+-Inf` (TODO: +- sign), `NaN`, and `+-0`.
While `f - 0.0`, `f * 1.0`, and `f / 1.0` are optimized away, you might be surprised to see that `f + 0.0` is not. (TODO: godbolt)
It turns out that IEEE 754 has a special rule for sums of values with equal magnitude but different signs:

> TODO: Quote IEEE 754 thing

Let's make a small table what this means for different operations:

| f | f + 0.0 | f - 0.0 | f + -0.0 | f - -0.0 |
| ---------- |
| +0.0 | +0.0 | +0.0 | +0.0 | +0.0 |
| -0.0 | **+0.0** | -0.0 | -0.0 | **+0.0** |

(Cases where the expression cannot be legally replaced by `f` are **bold**.)

> Yes, `f + 0.0` is slower than `f - 0.0`.

One might wonder if this whole `+0.0` vs `-0.0` business makes any difference.
Remember that `1.0 / +0.0 == Inf` and `1.0 / -0.0 == -Inf` and you can easily construct pathological cases:

```cpp
std::cout << std::max(3.0, 1.0 / (-1e-200 / 1e200 + 0.0)) << std::endl; // Inf
std::cout << std::max(3.0, 1.0 / (-1e-200 / 1e200 - 0.0)) << std::endl; // 3
```

Though overall, the usefulness of these special values can be doubted and if you are not actively expecting `Inf`s or `NaN`s you might be well advised to sprinkle a few `assert(isfinite(f))` across your code.


## Conclusion

At some point people tend to give up trying to understand floats and accept that they are "strange" or even "unreliable".
[And it's not exactly false](https://randomascii.wordpress.com/2013/07/16/floating-point-determinism/).
But they might be throwing the baby out with the bathwater.
It is true that floats involve a good amount of complexity but most rules are actually designed to _help_ making numerical algorithms reliable.

Don't forget that [IEEE 754 has a few useful guarantees](https://randomascii.wordpress.com/2017/06/19/sometimes-floating-point-math-is-perfect/).
For example, the five basic operations `+`. `-`, `*`, `/`, and `sqrt` are guaranteed to give "exact" results, which means the closest representable number corresponding to the current rounding mode.
This especially implies that everything that is representable will be computed exactly:

* `1.0 + 2.0 == 3.0`
* `3.0 / 4.0 == 0.75`
*  `sqrt(5.0625) == 2.25`

`float`s have 23 bit mantissa, meaning integer computations with `float` will be exact if input and output are at most a few millions.
`double` has enough precision that any `int32_t` computation will be exact.


## TODO

* TODO: links
* other interesting "missed" optimizations: `f * 0` (`-ffinite-math-only -fno-signed-zeros`), `f - f` (`-ffinite-math-only`), `f * 2` to `f + f`
* table with what can be optimized and which flags are required
* clamp instead of max