---
layout: post
current: post
cover:  assets/images/inline-skating.jpg
navigation: True
title: "Optimization without Inlining"
date: 2021-10-17 02:00:00
tags: [c++, optimization]
class: post-template
subclass: 'post'
author: philip
excerpt: "Even without inlining, the compiler does not always has to assume the worst case."
---

[Inlining](https://en.wikipedia.org/wiki/Inline_expansion) is one of the most important compiler optimizations.
We can often write abstractions and thin wrapper functions without incurring any performance penalty, because the compiler will expand the method for us at call site.

If a function is not inlined, conventional wisdom says that the compiler has to assume that the method can modify any global state and change the memory behind any pointer or reference that might have "escaped".

In this short post, I'll demonstrate exactly this effect.
Furthermore, we will see that even if a function is not inlined, as long as the implementation is visible, some optimizations are still performed and sometimes to great effect.

# Example

Let us consider this simple `test` function,

```cpp
// implemented in different TU or otherwise linked in
int foo(int n);

int test(int const& n) 
{
    auto sum = 0;
    sum += foo(n);
    sum += foo(n);
    return sum;
}
```

which results in the following [assembly](https://godbolt.org/z/oE5ozq1b4):

```cpp
test(int const&):
  push rbp
  push rbx
  push rax
  mov rbx, rdi
  mov edi, dword ptr [rdi] // load n from memory
  call foo(int)            // call foo
  mov ebp, eax
  mov edi, dword ptr [rbx] // load n from memory _again_
  call foo(int)            // call foo
  add eax, ebp
  add rsp, 8
  pop rbx
  pop rbp
  ret
```

Unsurprising, `foo` is called twice.
For all we know, it has important side effects that have to be executed twice.
Maybe a bit more subtle: `n` had to be reloaded from memory, because the first call to `foo` might have changed the memory.
(For example, `n` might be a reference to a global `int` that `foo` happens to increment.)

This is really the worst case for the compiler.
Nothing about the inner workings of `foo` is known.

On the other end of the spectrum, we have a small, known `foo`:

```cpp
int foo(int n)
{
    return n * n;
}

int test(int const& n) 
{
    auto sum = 0;
    sum += foo(n);
    sum += foo(n);
    return sum;
}
```

which results in the following [assembly](https://godbolt.org/z/6T6qsd79x):

```cpp
test(int const&):
  mov eax, dword ptr [rdi] // load n from memory
  imul eax, eax            // tmp = n*n
  add eax, eax             // return tmp + tmp
  ret
```

Not only is the call to `foo` inlined, further optimizations made sure that `n * n` is not computed twice.

# To Inline or not to Inline

If `foo` is part of a different TU, the compiler can obviously not inline the call.
Should you have enough patience, [link-time optimization](https://en.wikipedia.org/wiki/Interprocedural_optimization) might come to the rescue.
(It's usually really expensive, so I can only really recommend it for CI building release versions.)

However, there are many other reasons why `foo` is not inlined.
In general, inlining is a double-edged sword.
Each inlined function increases the pressure on the [instruction cache](https://en.wikipedia.org/wiki/CPU_cache) and makes the parent function harder to analyze.
More local variables mean more [register spills](https://en.wikipedia.org/wiki/Register_allocation).
All these can negatively affect performance and compilers have various heuristic to try to make good decisions.

Most small functions will get inlined by default.
As the complexity of the function body rises, this will stop at some point.
Recursion _usually_ prevents inlining, though [compilers are definitely able to inline recursive functions](https://godbolt.org/z/Ehdqcq37c) via [tail-call elimination](https://en.wikipedia.org/wiki/Tail_call).

We can artificially prevent inlining by declaring the function `__declspec(noinline)` (for msvc) or `__attribute__((noinline))` (for gcc/clang):

```cpp
__attribute__((noinline)) int foo(int n)
{
    return n * n;
}

int test(int const& n) 
{
    auto sum = 0;
    sum += foo(n);
    sum += foo(n);
    return sum;
}
```

which results in the following [assembly](https://godbolt.org/z/v5bPd99bo):

```cpp
test(int const&):
  mov edi, dword ptr [rdi] // load n from memory
  call foo(int)            // tmp = foo(n)
  add eax, eax             // return tmp + tmp
  ret
```

So, obviously `foo` was not inlined.
However, the compiler could still see and analyze it.
It came to the conclusion that `foo` does not read or modify global state.
`foo` was flagged as a [pure function](https://en.wikipedia.org/wiki/Pure_function).

Pure functions are nice.
Pure functions return the same result if given the same input.
Pure functions do not modify global state, they work solely on their input values.
Thus, `foo(n) + foo(n)` was simplified to `tmp = foo(n); tmp + tmp`, even without inlining.

# Fun with Loops

The difference becomes even larger with loops:

```cpp
// different TU
int foo(int n);

int test(int const& n) 
{
    auto sum = 0;
    for (auto i = 0; i <= n; ++i)
        sum += foo(n);
    return sum;
}
```

which results in the following [assembly](https://godbolt.org/z/4ErEvzecE):

```cpp
test(int const&):
  push rbp
  push r14
  push rbx
  mov r14, rdi
  mov edi, dword ptr [rdi]
  xor ebp, ebp
  test edi, edi // n < 0 case
  js .LBB0_3
  mov ebx, -1
.LBB0_2:         // loop body begin
  call foo(int)            // call foo
  add ebp, eax
  mov edi, dword ptr [r14] // reload n from memory
  inc ebx
  cmp ebx, edi
  jl .LBB0_2     // loop body end
.LBB0_3:
  mov eax, ebp
  pop rbx
  pop r14
  pop rbp
  ret
```

With no information, the compiler has to call `foo` (and load `n` from memory) in each iteration.
Contrast this with:

```cpp
__attribute__((noinline)) int foo(int n)
{
    return n * n;
}

int test(int const& n) 
{
    auto sum = 0;
    for (auto i = 0; i <= n; ++i)
        sum += foo(n);
    return sum;
}
```

which results in the following [assembly](https://godbolt.org/z/96Mxvsv9b):

```cpp
test(int const&):
  mov edx, dword ptr [rdi]
  test edx, edx
  js .L5 // special case n < 0
  mov edi, edx
  call foo(int)  // tmp1 = foo(n)
  imul edx, eax  // tmp2 = tmp1 * n
  add eax, edx   // return tmp2 + n
  ret
.L5:
  xor eax, eax // return 0
  ret
```

So gcc is able to optimize the whole thing to basically `foo(n) * (n + 1)`, without inlining.
Funnily enough, clang tries (and fails) to be clever with lots of SIMD.

# Conclusion

This is not a long post, but it shows that while inlining is a very important optimization, a non-inlined function is not the end of ~~the world~~ optimization.
As long as the function implementation is visible, compilers can and will analyze them so that they don't have to assume the worst case.
This, in turn, re-enables many optimizations such as [value numbering](https://en.wikipedia.org/wiki/Value_numbering), [common subexpression elimination](https://en.wikipedia.org/wiki/Common_subexpression_elimination), and [loop-invariant code motion](https://en.wikipedia.org/wiki/Loop-invariant_code_motion).

PS: gcc and clang have a variety of [function attributes](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html) that can be used to enable optimizations, even if the implementation is in a different TU.
The two most important ones are `__attribute__((const))` and `__attribute__((pure))`.

(_Title image from [unsplash](https://unsplash.com/photos/Y-VYK0SDLxs)_)
