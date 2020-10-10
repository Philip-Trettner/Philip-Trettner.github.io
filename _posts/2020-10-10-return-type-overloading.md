---
layout: post
current: post
cover:  assets/images/overload.jpg
navigation: True
title: "Overloading by Return Type in C++"
date: 2020-10-10 04:00:00
tags: [c++]
class: post-template
subclass: 'post'
author: philip
excerpt: 'Everyone overloads by argument types. But can you do return type?'
---

```cpp
// this is OK
std::string to_string(int i);
std::string to_string(bool b);

std::string si = to_string(0);
std::string sb = to_string(true);

// this is not OK
int from_string(std::string_view s);
bool from_string(std::string_view s);

int i = from_string("7");
bool b = from_string("false");
```

Overloading by argument types is a pretty straightforward feature of many imperative languages.
However, most of them don't support overloading by return types.
In particular, C++ does not.
For example, [clang complains](https://godbolt.org/z/ddWcE8):

```cpp
<source>:4:6: error: functions that differ only in their return type cannot be overloaded
bool from_string(std::string_view s);
~~~~ ^

<source>:3:5: note: previous declaration is here
int from_string(std::string_view s);
~~~ ^
```

So... what if I told you we actually _can_ overload by return type in C++?

...

By a slight misuse of user-defined conversion operators.


## A Proof-of-Concept

[Conversion operators can be user-defined](https://en.cppreference.com/w/cpp/language/cast_operator) in C++.
They allow us to add custom implicit or explicit conversion to our types.
These conversions themselves can also be overloaded, which leads us to a simple prototype:

```cpp
struct to_string_t
{
    std::string_view s;

    operator int() const;  // int  from_string(std::string_view s);
    operator bool() const; // bool from_string(std::string_view s);
};

int i = to_string_t{"7"};
bool b = to_string_t{"true"};
```

[Looking at godbolt](https://godbolt.org/z/f3chze), this compiles and calls the desired conversion operators.
An important point to note here is that the compiler needs to know the _target_ type for the conversion.
Thus, `auto i = to_string_t{"7"};` does not work as intended.
`i` will be of type `to_string_t` and not `int`.


## Packaging and Usage

We can achieve the original goal of an overloaded function by simply returning `to_string_t`:

```cpp
to_string_t from_string(std::string_view s) { return to_string_t{s}; }

int i = from_string("7");
bool b = from_string("true");
```

Alternatively, one can adhere to [almost always auto](https://herbsutter.com/2013/08/12/gotw-94-solution-aaa-style-almost-always-auto/) and write:

```cpp
auto i = int(from_string("7"));
auto b = bool(from_string("true"));
```

This technique also works when calling other functions:

```cpp
void foo(bool b, int i);

foo(from_string("false"), from_string("0"));
```

And even interacts properly with more complex, potentially templated objects:

```cpp
std::vector<int> vec;
std::map<int, bool> map;

vec.push_back(from_string("11"));
map[from_string("3")] = from_string("true");
vec.emplace_back(from_string("5"));
```

Note how [`emplace_back` is templated](https://en.cppreference.com/w/cpp/container/vector/emplace_back) and internally constructs an `int` from our `to_string_t`.

Finally, `if (cond)` tries to convert `cond` to `bool` and thus also works:

```cpp
if (from_string("true"))
    on_true();
else
    on_false();
```

These examples can be seen in action [at godbolt](https://godbolt.org/z/1x5de4).


## Where It Doesn't Work

While we can achieve overloading by return type in many cases using the conversion operator technique, it doesn't always apply.
As previously mentioned, the compiler needs to know the target type to choose the proper conversion operator and will not convert unless forced to.
We already saw the simplest case where no conversion is applied:

```cpp
auto i = to_string("7");
// decltype(i) is to_string_t, not int
```

Another problematic case arises when paired with normal overloading that leads to ambiguities:

```cpp
void bar(int);
void bar(bool);

bar(from_string("true"));
```

Which results in:

```cpp
error: call to 'bar' is ambiguous
bar(from_string("true"));
^~~
<source>:41:6: note: candidate function
void bar(int);
     ^
<source>:42:6: note: candidate function
void bar(bool);
     ^
```

Similarly, this means that `std::cout << from_string("2") << std::endl;` does not work.
(The error message for that is slightly ghastly as we have at least 16 candidate overloads.)

Finally, only one user-defined conversion can be applied implicitly, so the following [doesn't work](https://godbolt.org/z/Gxq1rb):

```cpp
struct bar 
{
    bar(int i);
};

void test_bar(bar b);

test_bar(from_string("3"));
```

The compiler only tries to directly convert `to_string_t` to `bar`:

```cpp
<source>:22:5: error: no matching function for call to 'test_bar'
    test_bar(from_string("3"));
    ^~~~~~~~
<source>:18:6: note: candidate function not viable: no known conversion from 'to_string_t' to 'bar' for 1st argument
void test_bar(bar b);
     ^
```

All these cases can be resolved by explicitly adding a cast to the desired type, e.g. `int(to_string("10"))`.


## Extensibility

One important aspect of normal function overloading in C++ is the extensibility of the overload set.
Independent authors and libraries can add to the same overload set simply by providing a function with the proper name in the same namespace.
We might also add to the overload set via [argument-dependent lookup](https://en.cppreference.com/w/cpp/language/adl), though if this should be considered feature or bug is slightly controversial.

In its base form, our conversion operator approach is not extensible.
User-defined conversion functions must be member functions and we cannot add members to other classes post-hoc.
Thus, if our library defines

```cpp
struct to_string_t
{
    std::string_view s;

    operator int() const;  // int  from_string(std::string_view s);
    operator bool() const; // bool from_string(std::string_view s);
};
```

Then this means "overload for return types `int` and `bool`" and other libraries / files cannot add to that.

There is a way to fix this and add extensibility.
This will add some implementation complexity and for more specialized use cases, extensibility might not actually be desired.
However, I would argue that `from_string` should be designed with extensibility in mind.

> Note: the rest of this section focuses more on metaprogramming and API design than the return-type overloading.
> You can skip to the next section to see the final version.

The solution here is that conversion functions can be templated.
We will use that to delegate the conversion to a template specialization, which is then properly extensible:

```cpp
template <class T>
struct to_string_impl 
{
    static_assert(always_false<T>, "conversion to T not supported");
};

struct to_string_t
{
    std::string_view s;

    template <class T>
    operator T() const { return to_string_impl<T>::from_string(s); }
};

to_string_t from_string(std::string_view s) { return to_string_t{s}; }
```

The conversion in `to_string_t` is now templated and always calls `to_string_impl<T>::from_string(s)`.
`to_string_impl<T>` is a class template that is specialized for all supported conversions.
Should a non-supported conversion be called, the [`always_false<T>` produces a nice(-ish) error message](/blog/2020/10/03/always-false).
We can now add our supported conversions via:

```cpp
template <>
struct to_string_impl<int>
{
    static int from_string(std::string_view s);
};

template <>
struct to_string_impl<bool>
{
    static bool from_string(std::string_view s);
};
```

And similarly, other authors or the end user can add conversions for custom types.
Sometimes, it is useful to conditionally add conversions.
For example `my_range<T>` might only be supported if `T` itself supports `from_string`.
Thus, it is customary to add a second template argument to the base template:

```cpp
template <class T, class = void>
struct to_string_impl 
{
    static_assert(always_false<T>, "conversion to T not supported");
};
```

This enables our imaginary end user to write:

```cpp
template <class T>
struct to_string_impl<my_range<T>, std::enable_if_t<has_from_string<T>>>
{
    static my_range<T> from_string(std::string_view s); // e.g. "[1, 2, 3]"
};
```

The partial specialization is only "active", if `T` itself satisfies `has_from_string<T>`.
(This, of course, is an example of [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae)).

Such a `has_from_string<T>` might look like this:

```cpp
template <class T>
auto impl_has_from_string(int) -> decltype(
    to_string_impl<T>::from_string(std::declval<std::string_view>()), 
    std::true_type{});

template <class T>
std::false_type impl_has_from_string(char);

template <class T>
constexpr bool has_from_string = decltype(impl_has_from_string<T>(0))::value;
```

Here, we use [Expression SFINAE](https://en.cppreference.com/w/cpp/language/sfinae#Expression_SFINAE) to disable the first `impl_has_from_string` overload if `to_string_impl<T>::from_string` does not exist.
`impl_has_from_string` itself is overloaded on `int` and `char` and called via `impl_has_from_string<T>(0)`.
This is a cheap way to say "try the `int` overload first and if it doesn't apply, take the `char` overload".
However, if we try to check `has_from_string<T>` for some type that has no `from_string`, we trigger the `static_assert(always_false<T>);` in the base template.
Thus, we move the `static_assert` to `to_string_t::operator T()` (see next section).

Note that the templated `to_string_impl` class is not the only option.
We could also use [tag dispatch](https://arne-mertz.de/2016/10/tag-dispatch/) or even normal overloading, e.g. by delegating to (a user-extensible) `void convert_to(std::string_view s, T& v)` that is overloaded on the second parameter.


## Final Version

For reference, our extensible and checkable version of return-type overloading in C++:

```cpp
// base template, specialize and provide a static from_string method
template <class T, class = void>
struct to_string_impl 
{
};

namespace detail // hide impl detail
{
template <class T>
auto has_from_string(int) -> decltype(
    to_string_impl<T>::from_string(std::declval<std::string_view>()), 
    std::true_type{});

template <class T>
std::false_type has_from_string(char);
}

// check if T has a from_string
template <class T>
constexpr bool has_from_string = decltype(detail::has_from_string<T>(0))::value;

// return-type overload mechanism
struct to_string_t
{
    std::string_view s;

    template <class T>
    operator T() const 
    {
        static_assert(has_from_string<T>, "conversion to T not supported");
        return to_string_impl<T>::from_string(s); 
    }
};

// convenience wrapper to provide a "return-type overloaded function"
to_string_t from_string(std::string_view s) { return to_string_t{s}; }
```

Anyone can register new types, optionally using SFINAE to conditionally support them:

```cpp
template <>
struct to_string_impl<int>
{
    static int from_string(std::string_view s);
};

template <>
struct to_string_impl<bool>
{
    static bool from_string(std::string_view s);
};

template <class T>
struct my_range { /* ... */ };

template <class T>
struct to_string_impl<my_range<T>, std::enable_if_t<has_from_string<T>>>
{
    static my_range<T> from_string(std::string_view s);
};
```

`has_from_string<T>` can be used to test (at compile time) if a `from_string` is available for a certain type:

```cpp
static_assert(has_from_string<int>);
static_assert(!has_from_string<char>);
static_assert(has_from_string<my_range<int>>);
static_assert(!has_from_string<my_range<float>>);
```

Finally, we still retain the original usage that looks like a return-type overloaded function:

```cpp
int i = from_string("7");
bool b = from_string("true");
my_range<int> r = from_string("[0, 1, 2]");
```

As always, [a godbolt link to back up my claims](https://godbolt.org/z/chfGa1).


## Summary

Overloading by argument types is ubiquitous in modern imperative languages but overloading by return type is usually not supported.
However, we can emulate it in C++ by (mis)using [user-defined conversion operators](https://en.cppreference.com/w/cpp/language/cast_operator).
As long as the target type is known, the proper "overload" is selected.
The basic version is simple:

```cpp
struct to_string_t
{
    std::string_view s;

    operator int() const;  // int  from_string(std::string_view s);
    operator bool() const; // bool from_string(std::string_view s);
};

to_string_t from_string(std::string_view s) { return to_string_t{s}; }
```

By default, this solution does not have the extensibility of normal by-argument-type overloading.
However, we can restore it via a templated conversion operator that delegates to a templated class that can be specialized.
In the process, we can also define a `has_from_string<T>` to help with diagnostics or SFINAE.


(_Title image from [pixabay](https://pixabay.com/photos/sign-street-road-road-signs-2454791/)_)
