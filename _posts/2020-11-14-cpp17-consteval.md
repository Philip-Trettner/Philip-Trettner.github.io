---
layout: post
current: post
cover:  assets/images/surveillance.jpg
navigation: True
title: "consteval in C++17"
date: 2020-11-14 02:00:00
tags: [c++]
class: post-template
subclass: 'post'
author: philip
excerpt: "Forcing compile time evaluation in C++17"
---

A common misconception is that `constexpr` functions are evaluated during compilation and not during runtime.
In reality, a `constexpr` function makes it _possible_ to be evaluated at compile time without guaranteeing it.

The rest of this post motivates why `constexpr` is sometimes not enough and how the following snippet can be used for a stronger compile-time evaluation guarantee without jeopardizing the ability to pass runtime values:

```cpp
template <auto V>
static constexpr auto force_consteval = V;
```

----

Consider the following `stringhash` function:

```cpp
constexpr size_t stringhash(char const* s) 
{
    size_t h = 0;
    while (*s) 
    {
        h = h * 6364136223846793005ULL + *s + 0xda3e39cb94b95bdbULL;
        s++;
    }
    return h;
}
```

This function can be used to compute string hashes at compile time:

```cpp
static_assert(stringhash("hello world") != 0);
```

Without `constexpr`, we would get:

```cpp
error: non-constant condition for static assertion
   14 | static_assert(stringhash("hello world") != 0);
      |               ~~~~~~~~~~~~~~~~~~~~~~~~~~^~~~
```

When the compiler is forced to, `constexpr` functions can be evaluated at compile time.
If not forced, there is no guarantee:

```cpp
size_t test()
{
    return stringhash("hello world");
}
```

With `-O0`, [neither clang nor gcc bother to evaluate the call at compile time](https://godbolt.org/z/K5xP78).
Fortunately, [with optimizations enabled, they actually do](https://godbolt.org/z/zGxdqY):

```cpp
test():
  movabsq $-9068320177771933951, %rax
  ret
```

However, this strongly depends on how the `constexpr` function is written.
A small variation, e.g. a recursive implementation, can be enough to change the compiler's mood:

```cpp
constexpr size_t stringhash(char const* s) 
{
    if (!*s)
        return 0;
    return stringhash(s + 1) * 6364136223846793005ULL + *s + 0xda3e39cb94b95bdbULL;
}
```

Now, [clang does not feel to be obligated to optimize it anymore](https://godbolt.org/z/cfjrb8), even at `-O2` or `-O3`.

In C++20, [we could use `consteval`](https://en.cppreference.com/w/cpp/language/consteval) to specify that `stringhash` must always produce a compile time constant expression.
Note that this would mean that we cannot use `stringhash` with runtime values anymore.

One way to force compile time evaluation is to declare a `constexpr` variable:

```cpp
size_t test()
{
    constexpr auto h = stringhash("hello world");
    return h;
}
```

This guarantees that `h` is initialized by a constant expression, which in turn means [compile time evaluation in practice](https://godbolt.org/z/K1qoTE), even for `-O0`.

Declaring additional `constexpr` variables for every call makes this quite cumbersome to use.
By using [variable templates](https://en.cppreference.com/w/cpp/language/variable_template) and [`auto` non-type template parameters](https://en.cppreference.com/w/cpp/language/template_parameters#Non-type_template_parameter), there is another, quite elegant solution:

```cpp
template <auto V>
static constexpr auto force_consteval = V;
```

Which can be used as:

```cpp
size_t test()
{
    return force_consteval<stringhash("hello world")>;
}
```

The reason I like this solution is because it enforces compile-time evaluation in `-O0` and `-O2` and [does not even introduce additional symbols](https://godbolt.org/z/nrozxb).
If `force_consteval` were not a `static` variable but rather `inline`, a function, or a type, then symbols might get emitted to ensure the value has the same address in each translation unit.
This would have a negative impact on binary size, especially if many different values are used across the program.

The `auto` means that all supported non-type template parameters are supported, e.g. any integral type, enum, or pointers.
With C++20 we also get floating-point types and literal types (including "user-defined `constexpr` classes").

This solution is still a bit verbose.
In C++17, it could be hidden behind a macro:

```cpp
#define STRINGHASH(str) force_consteval<stringhash(str)>
```

In C++20, we could [use custom literals](https://ctrpeach.io/posts/cpp20-string-literal-template-parameters/) to build a `stringhash<"hello world">`.

(_Title image from [pixabay](https://pixabay.com/photos/binoculars-watch-observation-2474698/)_)
