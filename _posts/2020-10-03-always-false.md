---
layout: post
current: post
cover:  assets/images/wrong-way.jpg
navigation: True
title: "always_false<T>"
date: 2020-10-03 04:00:00
tags: [c++]
class: post-template
subclass: 'post'
author: philip
excerpt: 'The working version of static_assert(false);'
---

```cpp
template <class T>
struct foo
{
    static_assert(false, "must use correct specialization");
};

template <>
struct foo<int>
{
    // valid use of foo<T> ...
};
```

Sometimes, good intentions are punished by [the compiler](https://godbolt.org/z/YdK3vc):

```
<source>:4:19: error: static assertion failed: must use correct specialization
    4 |     static_assert(false, "must use correct specialization");
      |                   ^~~~~
```

To be fair, it's just doing [its job](https://en.cppreference.com/w/cpp/language/static_assert):

> If `bool_constexpr` returns `true`, this declaration has no effect. Otherwise a compile-time error is issued, and the text of message, if any, is included in the diagnostic message.

Our intention was to trigger the static assertion only when `foo<T>` is instantiated with a non-supported type.
However, while processing declarations, the compiler sees `static_assert(false, ...)`, which it can immediately evaluate and thus issues an error.

In today's (rather short) post, we will use [type-dependent expressions](https://en.cppreference.com/w/cpp/language/dependent_name#Type-dependent_expressions) to realize our intentions.


## A simple `always_false<T>`

Our initial attempt does not work because the compiler can immediately evaluate the condition.
Just making the expression more complex is [not a solution](https://godbolt.org/z/n1oMxa):

```cpp
static_assert(sizeof(int) == 3, "must use correct specialization");
```

Still generates an error immediately.

> Side note: am I the only on who constantly tries to write `static_assert<cond>` instead of `static_assert(cond)`?

So, how can we "defer" the evaluation until the class is actually instantiated?

Using a similar solution to what we did in [the `dont_deduce<T>` post](/blog/2020/09/26/dont-deduce): taking away the compiler's ability to reason about an expression without an actual type for `T`.
The technical term for what we need is a [type-dependent expression](https://en.cppreference.com/w/cpp/language/dependent_name#Type-dependent_expressions).
Instead of `static_assert(false)`, you sometimes see the following patterns:

```cpp
template <class T>
struct foo
{
    static_assert(sizeof(T) != sizeof(T));
    static_assert(sizeof(T) < 0);
    static_assert(sizeof(T) + 1 == 0);
    static_assert(false && sizeof(T));
};
```

Now, the conditions formally depend on `T`, even if their actual value will always be false.
Still, this is enough to [silence the compiler](https://godbolt.org/z/Eb3PTe).
Only an actual instantiation will trigger the [error](https://godbolt.org/z/jr3af8):

```cpp
foo<bool> f;

// leads to:
<source>: In instantiation of 'struct foo<bool>':
<source>:13:11:   required from here
<source>:4:25: error: static assertion failed: must use correct specialization
    4 |     static_assert(false && sizeof(T), "must use correct specialization");
      |                   ~~~~~~^~~~~~~~~~~~
```

In my libraries, I usually formulate this as [a small helper](https://godbolt.org/z/5nYhc6) for consistency and clarity of intent:

```cpp
template <class... T>
constexpr bool always_false = false;
```

which is then used as:

```cpp
template <class T>
struct foo
{
    static_assert(always_false<T>, "must use correct specialization");
};
```

The definition of `always_false` is variadic so that multiple types can be provided.


## Use with `if constexpr`

I have two main use cases for `always_false<T>`.

The first is the already mentioned class template specialization when the "base case" is not supported.
A solution I sometimes see is to only declare but not define the template:

```cpp
template <class T>
struct foo; // only define the specializations
```

However, the [resulting error message](https://godbolt.org/z/dT9EhE) will confuse users of your API:

```cpp
<source>:10:11: error: aggregate 'foo<bool> f' has incomplete type and cannot be defined
   10 | foo<bool> f;
      |           ^
```

The second use case is in combination with `if constexpr`:

```cpp
template <class T>
void foo()
{
    if constexpr (std::is_same_v<T, int>)
    {
        // handle int case
    }
    else if constexpr (std::is_same_v<T, float>)
    {
        // handle float case
    }
    // ... other cases
    else
    {
        static_assert(false, "T not supported");
    }
}
```

This has the [exact same problem](https://godbolt.org/z/51Gbde): the static assertion triggers even without any use or instantiation of `foo`.
Luckily it also has the same solution: `static_assert(always_false<T>, "T not supported");`


## Summary

`static_assert(false)` always immediately triggers (unless `#ifdef`d) even if our intention is only to trigger on wrong instantiations.

The solution is to use [type-dependent expressions](https://en.cppreference.com/w/cpp/language/dependent_name#Type-dependent_expressions) and make the condition depend on our template parameters.
The main use cases are templated classes, where the "base case" should be forbidden and only "approved" specializations are allowed, and (chained) `if constexpr`, where we want to communicate unsupported cases via static assertions.

Popular ad-hoc solutions include `sizeof(T) + 1 == 0` or `false && sizeof(T)`.
I usually prefer a simple helper that clearly communicates intent:

```cpp
template <class... T>
constexpr bool always_false = false;
```

Discussion and comments on [reddit](https://www.reddit.com/r/cpp/comments/j4gsj4/always_falset/).

(_Title image from [pixabay](https://pixabay.com/photos/sign-street-road-road-signs-2454791/)_)
