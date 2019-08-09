---
layout: post
current: post
cover:  assets/images/zero-wall.jpeg
navigation: True
title: Special Treatment for Literal Zero
date: 2019-08-01 02:00:00
tags: [c++, fun]
class: post-template
subclass: 'post'
author: philip
excerpt: 'Or: how to make "foo(0)" and "foo(1)" call different functions'
---

## TL;DR:

It is possible in C++ to make `foo(0)` and `foo(1)` call different functions.
We used that to design an API where `x < 0` calls an optimized check without branching.

Trigger warning: this post contains mild forms of C++ gore.


## A Tale of `0` and `1`

I recently stumbled upon the little fact that just because `x < 1` compiles you cannot assume that `x < 0` would as well ([godbolt](https://godbolt.org/z/BJIDFx)).

```cpp
struct X { };
bool operator<(X, long);
bool operator<(X, void*);

X x;
x < 1; // fine
x < 0; // ambiguous overload
```

Let's start simple.
`0` and `1` are integer literals and without any suffix their type is good ol' `int`.
([Lack of a suffix alone is not a guarantee though.](https://godbolt.org/z/z7YlBy))
`int`s can be promoted to `long`s which is why `x < 1` happily calls the first overload.
So far so good.

Back in the days (technical term for the period of time before C++11), there was no `nullptr`.
People either used `0` or `NULL` ([a fancy macro](https://en.cppreference.com/w/cpp/types/NULL)).
The reason this worked was because there is a [special rule](https://en.cppreference.com/w/cpp/language/implicit_conversion#Pointer_conversions) that `0` can be converted to any pointer and yields the null pointer.

`x < 0` is thus ambiguous because both overloads are valid and they are equally good (both are one conversion away).


## An Idea Forms

This little curious gem led me to rethink a recent API design problem we faced.

We were writing a custom 256 bit integer type, basically a `BigInteger` but with a fixed representation using 4 `uint64_t`s internally.
This was performance sensitive code and we wanted everything to be maximally efficient.

Our `i256` type uses two's complement and the check `x < 0` can be done by simply testing if the highest bit is set.
However, `x < 1` or `x < y` was a bit more expensive because you basically have to test `x - y < 0`.

In our initial design we opted for a `is_below_zero(i256)` function that performs the optimized test and `operator<(i256, i256)` that does the generic test.
Unfortunately, people tend to write `x < 0` and I can't blame them because it's more readable and more concise than `is_below_zero(x)`.

Looping back to our initial observation:
If it is possible to generate a compile error for `x < 0` but not for `x < 1` then maybe it is also possible to redirect `x < 0` to a different function.

> What if we could automatically (and statically) use `is_below_zero(x)` whenever `operator<` is called with a literal `0` and otherwise fall back to the generic code without using a runtime branch?


## The Initial Design

Usually, whenever we want to check if something would compile without actually introducing a compile error, we reach for [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae).
The idea would be to disable `operator<(long)` for `0`.
In our case, this doesn't work because SFINAE usually deals in types where `0` and `1` are not distinguishable anymore.

However, we don't actually need to disable `operator<(long)`, we just need to break the ambiguity.
If there were a way to make `operator<(long)` a worse candidate than `operator<(void*)` (for `0`) then this meets our requirements.

Lo and behold, there is a way!

User conversions are [famously](https://stackoverflow.com/questions/44086269/why-does-my-variant-convert-a-stdstring-to-a-bool) [known](https://stackoverflow.com/questions/44021989/implicit-cast-from-const-string-to-bool) for being "weaker" than standard conversions.

```cpp
struct not_literal_zero { not_literal_zero(int); };
using literal_zero = void*;

struct X { };
bool operator<(X, not_literal_zero); // (1)
bool operator<(X, literal_zero);     // (2)

X x;
x < 1; // calls (1)
x < 0; // calls (2)
```

`1` cannot be converted to `void*` but has an implicit conversion to `not_literal_zero`.
`0` can be converted to both but `void*` is a standard conversion while `not_literal_zero` a user-provided one, making (2) the unambiguous call target ([godbolt](https://godbolt.org/z/Ke8rWa)).


## Banishing Footguns

Are we done yet?

Well... yes, and no.

We achieved our initial goal of making `x < 0` call our optimized version while `x < 1` and `x < y` call the generic version.
APIs should not be needlessly surprising but we're good law-abiding citizen:
Both versions are semantically equivalent for `x < 0`.
The call is statically resolved, thus no performance overhead in any case and a performance win for the common `x < 0` test.

However, we introduced a few unnecessary [footguns](https://en.wiktionary.org/wiki/footgun):

1. `x < nullptr` compiles
2. `x < &x` compiles
3. `x < "y tho?"` almost compiles

The third case would compile if we use `void const*` instead of `void*`, something we might be tempted to do due to const-correctness.

Let's fix the first case.
This is actually pretty simple because `nullptr` has a special type `std::nullptr_t`.
We cannot add an overload with `std::nullptr_t` as that would make `x < 0` ambiguous again.
However, we can disable it with SFINAE:

```cpp
#include <type_traits>

template<class T, 
         class = std::enable_if_t<std::is_same_v<T, std::nullptr_t>>>
bool operator<(X, T) = delete;
```

For `x < 0`, `T` will be `int` and this overload is disabled via `std::enable_if`.
For `x < nullptr`, `T` is deduced to be `std::nullptr_t` and the overload is valid.
It requires no additional conversion and thus matches better than our `void*` overload.
The compile error is not ideal (`use of deleted function 'operator<'`) and could be improved with a `static_assert` if desired.

The final issue is that `x < &x` compiles.
The reason is that many pointer types can be converted to `void*`, which is a pretty general type.
While we cannot fix all corner cases, we can make it incredibly hard to accidentally trigger it.

First, we change `void*` into a [pointer-to-data-member](https://en.cppreference.com/w/cpp/language/pointer#Pointers_to_data_members), let's say `X* X::*` (a pointer to a member of `X` that is itself a pointer to `X`).
`0` can still initialize this pointer, but the only other way would be `&X::m` (where `m` is an actual member of `X` of type `X*`) or via explicit casts.

If you really don't want to take any chances, you can also reach for a [Voldemort Type](http://videocortex.io/2017/Bestiary/#-voldemort-types):

```cpp
auto hidden_ptr_to_member()
{
    struct H { };
    H* H::* p = nullptr;
    return p;
}

using literal_zero = decltype(hidden_ptr_to_member());

bool operator<(X, literal_zero);
```

Outside of `hidden_ptr_to_member` there is no syntax to name the inner `struct H`, thus making it a _type-that-cannot-be-named_.
The only way to actually refer to it is by using `decltype(...)`.
Apart from the inability to name it, it is just a pointer-to-data-member and thus suitable for our `x < 0` overload.

For a final touch and maybe as an appeal to our inner humanity, we move `hidden_ptr_to_member` to `namespace impl`.
`impl::` and `detail::` namespaces are basically C++ bro code for "I have to expose this because of C++ rules but if you actively use anything inside it, you're on your own".


## Summary

And with that, we have our [final version](https://godbolt.org/z/r1sYsE):

```cpp
#include <type_traits>

namespace impl
{
auto hidden_ptr_to_member()
{
    struct H { };
    H* H::* p = nullptr;
    return p;
}
}

struct not_literal_zero { not_literal_zero(int); };
using literal_zero = decltype(impl::hidden_ptr_to_member());

struct X { };
bool operator<(X, not_literal_zero); // (1)
bool operator<(X, literal_zero);     // (2)

template<class forbid_nullptr, 
         class = std::enable_if_t<std::is_same_v<forbid_nullptr, std::nullptr_t>>>
bool operator<(X, forbid_nullptr) = delete; // (3)

X x;
x < 1;       // calls (1)
x < 0;       // calls (2)
x < nullptr; // calls (3) which is deleted
x < &x;      // no matching call
```

`x < 0` and `x < 1` call different functions and we can safely optimize the `x < 0` case.
With creative use of some slightly more arcane C++ rules we were able to remove a few pitfalls (like being able to call `x < y` with `nullptr` or various other pointer types).

I probably have to say that I consider this thing more in the category of "fun with C++" than "production-ready code", so please use your own judgement before adopting it.
A possible middle ground could be 
```cpp
[[deprecated("use is_below_zero(X) instead")]]
bool operator<(X, literal_zero);
```