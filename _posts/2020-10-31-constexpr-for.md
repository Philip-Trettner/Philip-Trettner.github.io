---
layout: post
current: post
cover:  assets/images/loop.jpg
navigation: True
title: "Approximating 'constexpr for'"
date: 2020-10-31 02:00:00
tags: [c++]
class: post-template
subclass: 'post'
author: philip
excerpt: "'if constexpr' is awesome, but can we do 'constexpr for'?"
---

In the ancient times, template metaprogramming was all about recursive templates and template specialization.
With each version after C++11, the set of tools for convenient compile-time programming became larger and larger.
Many compile-time computations can now be solved elegantly via [various `constexpr` constructs](https://en.cppreference.com/w/cpp/language/constexpr).
[Fold expressions](https://en.cppreference.com/w/cpp/language/fold) massively simplify operations on variadic templates.

Traditionally, switching between different implementations based on compile-time computed properties has be done via static functions in template specializations, [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae), or [tag dispatch](https://arne-mertz.de/2016/10/tag-dispatch/).
These require writing at least one function per implementation.
Since C++17, [constexpr if](https://en.cppreference.com/w/cpp/language/if) enables us to coalesce different implementation in the same (templated) function.
This has many advantages, from improving compile times to reducing DRY violations by reusing parts of the implementation and not having to repeat the function signature.

One notable missing feature: `constexpr for`.

While we can use `for` loops in `constexpr` functions, the loop variable cannot be used in a constant expression itself:

```cpp
template <int N>
constexpr std::array<float, N> make_data();

template <int Start, int End>
constexpr float data_sum()
{
    float sum = 0;
    for (auto i = Start; i < End; ++i)
    {
        auto data = make_data<i>();
        for (auto v : data)
            sum += v;
    }
    return sum;
}
```

Here, `make_data` produces data with a predefined size `N`.
In `data_sum`, we want to sum up all provided data for a certain range of values `N`.
`Start` and `End` are known compile-time, so [why does `make_data<i>()` not work](https://godbolt.org/z/K9rddb)?

Well, `i` changes its values in each iteration, which changes the type of `data`, which means we need different code for the loop body in each iteration.
Intuitively, we long for a `constexpr for` that does a kind of [loop unrolling](https://en.wikipedia.org/wiki/Loop_unrolling).

Note that in this example, the `constexpr for` would even be useful if the functions are not `constexpr`: `Start` and `End` are compile-time constant and the roadblock here is that the type of `data` depends on `i`.

In this post I will present three approximations to `constexpr for` that solve some common use cases.


## Classical Integral For

First, let's fix the introductory example.
In the absence of a better name, I call these "integral for" loops:

```cpp
for (auto i = Start; i < End; i += Inc)
    f(i);
```

Where `Start`, `End`, and `Inc` are compile-time constant integral types, e.g. `int`s.

We need to solve two problems:
First, we must ensure that `i` is also a compile-time constant.
Second, a different loop body must be instantiated for each `i`, otherwise it's impossible to use types that depend on `i` inside the body.

The solution is surprisingly simple using `if constexpr`, recursive instantiation, and [std::integral_constant](https://en.cppreference.com/w/cpp/types/integral_constant):

```cpp
template <auto Start, auto End, auto Inc, class F>
constexpr void constexpr_for(F&& f)
{
    if constexpr (Start < End)
    {
        f(std::integral_constant<decltype(Start), Start>());
        constexpr_for<Start + Inc, End, Inc>(f);
    }
}
```

Since C++17, we can use `auto` as a non-type template parameter to support different loop variable types.
The code should be largely self-explanatory but the use of `std::integral_constant` might be non-obvious.
It should become clear if we take a look at how this solves our initial example:

```cpp
template <int N>
constexpr std::array<float, N> make_data();

template <int Start, int End>
constexpr float data_sum()
{
    float sum = 0;
    constexpr_for<Start, End, 1>([&sum](auto i){
        auto data = make_data<i>();
        for (auto v : data)
            sum += v;
    });
    return sum;
}
```

First of all, [this works as intended and unrolls the loop](https://godbolt.org/z/TPTxEW). 
Even in a full `constexpr` context, [it compiles and can be fully precomputed](https://godbolt.org/z/e3soa1).

So... why `std::integral_constant`?

Because that's our way to instantiate a different function for each iteration _and_ have a constant expression `i`.
We pass a generic lambda with `(auto i)` as parameter.
Thus, each different value for `i` causes a new instantiation as `std::integral_constant<int, 1>` and `std::integral_constant<int, 2>` are different types.
`make_data<i>()` works because `std::integral_constant<class T, T v>` has an implicit `constexpr` conversion to `T`.
It becomes clearer if you think about it as `make_data<decltype(i)::value>()`.
Of course, the shorter `make_data<i.value>()` would have worked as well as static members can also be accessed via dot syntax.

> In C++20, we could have used `[&sum]<int I>() { ... }` instead.


## Parameter Packs

Another common use case is iterating over a parameter pack:

```cpp
template <class... Args>
void print_all(Args const&... args)
{
    for (auto const& a : args)
        std::cout << a << std::endl;
}
```

But it doesn't actually work that way.
Same problem as before: each loop iteration needs a different instantiation as the types of `args` are heterogeneous.
The generic solution is almost trivial:

```cpp
template <class F, class... Args>
constexpr void constexpr_for(F&& f, Args&&... args)
{
    (f(std::forward<Args>(args)), ...);
}
```

Here, we used a [fold expression](https://en.cppreference.com/w/cpp/language/fold) of the comma operator to call `f` separately with each argument.
An empty parameter pack leads to zero invocations of `f`, in which case the pack expands to `void()`.
Note that the perfect forwarding here allows us to properly handle rvalues as well, e.g. passing a `unique_ptr` by value.
This can now be used as:

```cpp
template <class... Args>
void print_all(Args const&... args)
{
    constexpr_for([](auto const& v) {
        std::cout << v << std::endl;
    }, args...);
}
```

Again, [the compiler can unroll and properly call each function](https://godbolt.org/z/53ebaP).

> For more fold expression magic, I can wholeheartedly recommend [nifty fold expression tricks](https://foonathan.net/2020/05/fold-tricks/) by Jonathan Müller.

## Tuples and Tuple-Likes

`std::tuple`, `std::array`, and `std::pair` have specializations for [std::tuple_size](https://en.cppreference.com/w/cpp/utility/tuple/tuple_size).
This is also the recommended customization point when adding [structured binding](https://en.cppreference.com/w/cpp/language/structured_binding) support for non-trivial custom types.

We can easily define a `constexpr for` approximation for tuple-like types using our "integral for" helper from the beginning:

```cpp
template <class F, class Tuple>
constexpr void constexpr_for_tuple(F&& f, Tuple&& tuple)
{
    constexpr size_t cnt = std::tuple_size_v<std::decay_t<Tuple>>;

    constexpr_for<size_t(0), cnt, size_t(1)>([&](auto i) {
        f(std::get<i.value>(tuple));
    });
}
```

Which can then be used to iterate over tuple-like objects:

```cpp
constexpr_for_tuple([](auto const& v) {
        std::cout << v << std::endl;
    },
    std::make_tuple(1, 'c', true));
```


## Summary

While there is currently no `constexpr for` in C++, we can easily write a [std::for_each](https://en.cppreference.com/w/cpp/algorithm/for_each)-like approximation for it.
The important part is that each iteration has a unique instantiation to make it possible to use different types per iteration.
This can be elegantly handled by a generic lambda.

For the "integral for" loop, it may also be important to have access to the index as a constant expression, so it can be used as non-type template parameter.
This can be supported by using [std::integral_constant](https://en.cppreference.com/w/cpp/types/integral_constant) to pass the value.

In the end, we have three `constexpr for` approximations for different use cases:

* `for (auto i = Start; i < End; i += Inc)` ("integral for")
* `for (auto&& a : args...)` ("parameter pack for")
* `for (auto&& v : tuple_like_obj)` ("tuple-like for")

Of course, you should typically prefer a normal `for` loop as per-iteration instantiation will not be gentle on the compile time.
However, sometimes this is not possible because we either need the loop index as non-type template parameter or the variable type might change in each iteration.


## Further Reading

* There is official proposal [P1306](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1306r1.pdf) for `for...`
* [Thoughts on P1306](https://quuxplusone.github.io/blog/2019/02/28/expansion-statements/) by Arthur O'Dwyer
* [Boost.Hana](https://www.boost.org/doc/libs/1_65_1/libs/hana/doc/html/index.html), especially [hana::for_each](https://www.boost.org/doc/libs/1_63_0/libs/hana/doc/html/group__group-Foldable.html#ga2af382f7e644ce3707710bbad313e9c2)

(_Title image from [pixabay](https://pixabay.com/photos/rollercoaster-looping-amusement-801833/)_)
