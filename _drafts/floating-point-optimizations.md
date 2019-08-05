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
excerpt: 
---

Many posts have been written about elaborate magic involving IEEE 754 floating point numbers.
Some of my favorites include John Carmack's reverse square root (TODO: link), (TODO: Random Ascii Floating Stuff), and the reverse-Z depth test for rendering (TODO: link).

In this post I want to take a look at less flashy, more foundational things.
We're going over various small optimizations that we probably expected from our compiler but that are not actually legal or only work due to slightly arcane rules.

While this might apply to other languages as well, I'm most familiar with C++ which I'll be using for this post.
Because C++ is actually a standard (with a big chunk of it dedicated to specifying corner cases), we can actually get authorative answers for most of our questions!

We will furthermore assume that our C++ implementation uses:

* Two's complement for integers
* IEEE 754 for floating points

(This is currently implementation-defined behavior, but might change (TODO: link))

As always, it is highly instructive to look at the generated assembly for which we will use the excellent page by [Matt Godbolt](TODO) (TODO: link).


## The Curios Case of `f + 0.0`

Most programmers are taught that floating point math is neither commutative nor associative (TODO: link) (`1e9 + 1 - 1e9 == 0` vs. `1e9 - 1e9 + 1 == 1`).
The same goes for integers if overflows might be involved.

At some point people tend to give up trying to understand floats and accept that they are "strange" or even "unreliable".
And it's not exactly false. (TODO: insert links)
But they might be throwing the baby out with the bathwater. (TODO: correct idiom?)
It is true that floats involve a good amount of complexity but most rules are actually designed to _help_ making numerical algorithms reliable.
(TODO: is this paragraph ok?)

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
| +0.0 | +0.0 | +0.0 | +0.0 | **-0.0** |
| -0.0 | **+0.0** | -0.0 | -0.0 | -0.0 |

(Cases where the expression cannot be legally replaced by `f` are **bold**.)

One might wonder why this whole `+0.0` vs `-0.0` business is useful.
Remember that `1.0 / +0.0 == Inf` and `1.0 / -0.0 == -Inf` and you can easily construct pathological cases:

```cpp
std::cout << std::max(3.0, 1.0 / (-1e-200 / 1e200 + 0.0)) << std::endl; // Inf
std::cout << std::max(3.0, 1.0 / (-1e-200 / 1e200 - 0.0)) << std::endl; // 3
```


## TODO

* check spelling of carmack
* add links
* remove font size and line height from post-template kg-card-markdown p first-child
* check if justified is better
