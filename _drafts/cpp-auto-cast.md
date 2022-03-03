---
layout: post
current: post
cover:  assets/images/cast-iron.jpg
navigation: True
title: "auto_cast in C++"
date: 2022-03-02 02:00:00
tags: [c++, fun]
class: post-template
subclass: 'post'
author: philip
excerpt: "An almost magical cast."
---

A colleague and I were recently browsing the overview section of the [Odin programming language](https://odin-lang.org/) and came upon the [`auto_cast` operator](https://odin-lang.org/docs/overview/#auto-cast-operation):

```cpp
x: f32 = 123
y: int = auto_cast x
```

Struck by inspiration he said "there oughta be some piece of godless C++ code for that".

And indeed, in this post we will see how [overloading by return type](/blog/2020/10/10/return-type-overloading) leads to a passable imitation of that feature.

> Disclaimer: While we will work out several kinks in the implementation, this is definitely more of a "can I do that?" instead of a "should I do that?" post.

# The Basis

(Too) Many cast in C++ are implicit.
In particular, the Odin example would compile without issue.
Our running example will be slightly different and require a genuine `static_cast` to compile:

```cpp
void* p = nullptr;

// ERROR:
float* f = p;

// OK:
float* f = static_cast<float*>(p);

// Target of this post:
float* f = auto_cast(p);
```

The trick for [overloading by return type](/blog/2020/10/10/return-type-overloading) is to write a wrapper class with multiple [user-defined conversion operators](https://en.cppreference.com/w/cpp/language/cast_operator).
A simple implementation for `auto_cast` is thus:

```cpp
template <class From>
struct auto_cast
{
    From value;

    auto_cast(From v) : value(v) {}

    template <class To>
    operator To() 
    { 
        return static_cast<To>(value); 
    }
};
```

The expression `auto_cast(p)` will then use [CTAD](https://en.cppreference.com/w/cpp/language/class_template_argument_deduction) to deduce `From = void*`.
Assigning `float* f = auto_cast(p)` will instantiate `operator To()` with `To = float*`, which will then perform the appropriate `static_cast`.
Our `auto_cast` is (independently) generic in input and output, so two template parameters is the expected amount.

We solved the problem.
The post is over.

... but you can still stay a bit to see some improvements over this almost naive solution.


# Making Misuse Harder

The initial solution has a few problems that can lead to some unexpected behavior.

`auto` will not convert or "decay" our wrapper, so especially some styles like [almost always auto](https://herbsutter.com/2013/08/12/gotw-94-solution-aaa-style-almost-always-auto/) could litter the `auto_cast` type unexpectedly.
Furthermore, we have a single-argument constructor, which is implicit per default.
Values of type `From` will happily create `auto_cast` instances, especially on assignment.

```cpp
auto v = auto_cast(p);
// v is still auto_cast<T>
// and will happily try to convert wherever used

float* f = nullptr;
v = f; // calls implicit ctor
```

These issues are easily fixable:

```cpp
template <class From>
struct auto_cast
{
    explicit auto_cast(From v) : value(v) {}

    template <class To>
    operator To() &&
    { 
        return static_cast<To>(value); 
    }

private:
    From value;
};
```

We made the constructor `explicit` and added an [rvalue reference qualifier](https://en.cppreference.com/w/cpp/language/member_functions#ref-qualified_member_functions) to the conversion operator, so it can only be used on rvalues of `auto_cast`, which we have in `f = auto_cast(p)`, but not in `auto v = auto_cast(p); f = v;`.
For good measure, we also made `value` private.

# Better? Error Messages

```cpp
int i = auto_cast(p);
// clang 13:
// error: static_cast from 'void *' to 'int' is not allowed
// note: in instantiation of function template specialization 
//       'auto_cast<void *>::operator int<int>' requested here
```

This is arguably a good error message already, but it happens inside the template instantiation and is also not SFINAE-friendly.
We can guard the conversion operator itself by SFINAE and remove it when `From` is not convertible to `To`.
In C++20, this can be neatly done using [the `std::convertible_to` concept](https://en.cppreference.com/w/cpp/concepts/convertible_to):

```cpp
template <std::convertible_to<From> To>
operator To() &&
{ 
    return static_cast<To>(value); 
}
```

With which we now get:

```cpp
int i = auto_cast(p);
// clang 13:
// error: no viable conversion from 'auto_cast<void *>' to 'int'
// note: candidate template ignored: constraints not satisfied [with To = int]
// note: because 'std::convertible_to<int, void *>' evaluated to false
```


# TODO

* casting assignment!
* references
* other cast types

future work
* arrays?

TODO

(_Title image from [pixabay](https://pixabay.com/photos/drain-domain-domna-slag-2747070/)_)
