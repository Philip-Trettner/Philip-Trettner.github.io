---
layout: post
current: post
cover:  assets/images/lake_berryessa.jpg
navigation: True
title: Special Treatment for Literal Zero
date: 2019-07-31 02:00:00
tags: [c++, fun]
class: post-template
subclass: 'post'
author: philip
---

## TL;DR:

It is possible in C++ to make `foo(0)` and `foo(1)` call different functions.
We used that to design an API where `x < 0` calls an optimized check without branching.

Trigger warning: this post contains mild forms of C++ gore.


## A Tale of `0` and `1`

I recently stumbled upon the little fact that just because `x < 1` compiles, you cannot assume that `x < 0` would as well ([godbolt](https://godbolt.org/z/BJIDFx)).

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
People either used `0` or `NULL`, which is a [fancy macro](https://en.cppreference.com/w/cpp/types/NULL).
The reason this worked was because there is a [special rule](https://en.cppreference.com/w/cpp/language/implicit_conversion#Pointer_conversions) that `0` can be converted to any pointer and yields the null pointer.

`x < 0` is thus ambiguous because both overloads are valid and they are equally good (both are one conversion away).


## An Idea Forms

This little curious gem led me to rethink a recent API design problem I faced.

We were writing a custom 256 bit integer type, basically a `BigInteger` but with a fixed representation using 4 `uint64_t` internally.
This was performance sensitive code and we wanted everything to be maximally efficient.

Our `i256` type uses two's complement and the check `x < 0` can be done by simply testing if the highest bit is set.
However, `x < 1` or `x < y` was a bit more expensive because you basically have to test `x - y < 0`.

In our initial design we opted for a `is_below_zero(i256)` function that performs the optimized test and `operator<(i256, i256)` that does the generic test.
Unfortunately, people tend to write `x < 0` and I can't blame them because it's more readable and more concise than `is_below_zero(x)`.

Looping back to our initial observation:
If it is possible to generate a compile error for `x < 0` but not for `x < 1` then maybe it is also possible to redirect `x < 0` to a different function.

_What if we could automatically (and statically) use `is_below_zero(x)` whenever `operator<` is called with a literal `0` and otherwise use the generic code?_


## The Initial Design

Usually, whenever we want to check if something would compile without actually introducing a compile error, we reach for [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae).
The idea would be to disable `operator<(long)` for `0`.
In our case, this doesn't work because SFINAE usually deals in types where `0` and `1` are not distinguishable anymore.

However, we don't actually need to disable `operator<(long)`, we just need to break the ambiguity.
If there were a way to make `operator<(long)` a worse candidate for `0` than `operator<(void*)` then this meets our requirements.

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

The third case would compile if we would use `void const*` instead of `void*`, something we might be tempted to do due to const-correctness.

TODO: continue


## Summary

TODO


## TODO

TODO: middle ground: `[[deprecated("use ...")]]`
