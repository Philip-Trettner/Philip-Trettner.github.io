---
layout: post
current: post
cover:  assets/images/murder.jpg
navigation: True
title: "dont_deduce&lt;T&gt;"
date: 2020-09-26 02:00:00
tags: [c++, api design]
class: post-template
subclass: 'post'
author: philip
excerpt: 'Reducing API friction by a seemingly useless typedef'
---

```cpp
template <class T>
struct foo_t 
{
    using type = T;
};

template <class T>
using foo = typename foo_t<T>::type;
```

Now that's a pretty useless snippet.

... or is it??


## Controlling Type Deduction

Spoiler alert: it's not useless.

In particular, it allows us to control [template argument deduction](https://en.cppreference.com/w/cpp/language/template_argument_deduction) to a certain extent.

In my libraries, I usually define the typedef as follows:

```cpp
template <class T>
struct dont_deduce_t 
{
    using type = T;
};

template <class T>
using dont_deduce = typename dont_deduce_t<T>::type;
```

This clearly communicates our intent: we want to disable type deduction for a certain parameter.

> In C++20, the same functionality is provided in `<type_traits>` under [std::type_identity](https://en.cppreference.com/w/cpp/types/type_identity). (though I find this name significantly less clear in a function declaration)

Okay okay, not so fast.
What problem are we trying to solve here?


## Motivating Example: Vector Math

```cpp
template <class T>
struct vec3
{
    T x, y, z;
};

template <class T>
vec3<T> operator+(vec3<T> const& a, T b)
{
    return {a.x + b, a.y + b, a.z + b};
}
```

That looks like a reasonable definition of `operator+`, doesn't it?

Turns out, it doesn't provide the smooth API that we'd like to have.

```cpp
vec3<float> v = ...;
v = v + 3; // that'd be a cool API, right?
```

[GCC 10.2 politely refuses this code](https://godbolt.org/z/Yx4jr1) but not without a proper explanation:

```cpp
<source>:16:11: error: no match for 'operator+' (operand types are 'vec3<float>' and 'int')
   16 |     v = v + 3;
      |         ~ ^ ~
      |         |   |
      |         |   int
      |         vec3<float>
<source>:8:9: note: candidate: 'template<class T> vec3<T> operator+(const vec3<T>&, T)'
    8 | vec3<T> operator+(vec3<T> const& a, T b)
      |         ^~~~~~~~
<source>:8:9: note:   template argument deduction/substitution failed:
<source>:16:13: note:   deduced conflicting types for parameter 'T' ('float' and 'int')
   16 |     v = v + 3;
      |             ^
```

What happens is that `operator+` is called with a `vec3<float>` and `int`.
The compiler then tries to _deduce_ a `T` such that the signature `(vec3<T> const&, T)` is satisfied.
For the first argument it figures `T = float` might be a good match while for the second, `T = int` is the natural choice.
Thus it responds: `deduced conflicting types for parameter 'T' ('float' and 'int')`.

The compiler is only happy if the deductions for all arguments agree.
However, our intention was more along the lines of:
"Deduce `T` from `vec3<T> const&` and then try to convert `b` to `T`, preferably with an error if this doesn't work".

We _could_ make this work with additional template arguments and [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae):

```cpp
template <class T, class B, std::enable_if_t<std::is_convertible_v<B, T>, int> = 0>
vec3<T> operator+(vec3<T> const& a, B b);
```

However, in my opinion the superior solution is to "disable deduction" for `b` by turning its type into a so called [non-deduced context](https://en.cppreference.com/w/cpp/language/template_argument_deduction#Non-deduced_contexts):

```cpp
template <class T>
vec3<T> operator+(vec3<T> const& a, dont_deduce<T> b);
```

And voilà, now `v + 3` [just works](https://godbolt.org/z/jsnbMr).

As a non-deduced context, the second argument of `operator+` is not used for template type deduction.
Thus, only `vec3<T> const&` is matched against the `vec3<float>`, resulting in an unambiguous `T = float`.
After the typedef is resolved, we have `operator+(vec3<float> const&, float)` which is called with `vec3<float>` and `int`, which is perfectly fine as there obviously is a conversion from `int` to `float`.

If the second argument is not convertible (e.g. `v + "foo"`), we get [a nice error message](https://godbolt.org/z/onh7es):

```cpp
<source>:25:11: error: no match for 'operator+' (operand types are 'vec3<float>' and 'const char [4]')
   25 |     v = v + "foo";
      |         ~ ^ ~~~~~
      |         |   |
      |         |   const char [4]
      |         vec3<float>
<source>:17:9: note: candidate: 'vec3<T> operator+(const vec3<T>&, dont_deduce<T>) [with T = float; dont_deduce<T> = float]'
   17 | vec3<T> operator+(vec3<T> const& a, dont_deduce<T> b)
      |         ^~~~~~~~
<source>:17:52: note:   no known conversion for argument 2 from 'const char [4]' to 'dont_deduce<float>' {aka 'float'}
   17 | vec3<T> operator+(vec3<T> const& a, dont_deduce<T> b)
      |                                     ~~~~~~~~~~~~~~~^
```

This also highlights a subtle difference between the twe SFINAE and the `dont_deduce<T>` solution:
With SFINAE, the function overload does not really exist (though modern compilers still give [reasonable, though often confusing error messages](https://godbolt.org/z/h3K4TE)).
With `dont_deduce<T>`, the function exists and it's like calling a function with the wrong type of parameters.


## Other Useful Examples

That's all that I wanted to explain about `dont_deduce<T>`.
What follows are a few additional examples where it makes for a better API (in my opinion) if some arguments are not deduced.

### Clamping

```cpp
template <class T>
T clamp(T value, dont_deduce<T> min, dont_deduce<T> max)
{
    return value < min ? min : value > max ? max : value;
}

// otherwise this wouldn't work:
float v = ...;
v = clamp(v, 0, 1);
```

### Contains with Epsilon

```cpp
template <class T>
bool contains(sphere3<T> const& sphere, pos3<T> const& p, dont_deduce<T> eps)
{
    return distance(sphere.center, p) <= sphere.radius + eps;
}

// otherwise this wouldn't work:
sphere3<float> s = ...;
pos3<float> p = ...;
if (contains(s, p, 1e-5)) ...;
```

### Queries with Defaults

```cpp
template <class T>
T get_property_or(property_handle<T> prop, dont_deduce<T> default_val);

// otherwise this wouldn't work:
property_handle<std::string_view> p;
auto s = get_property_or(p, "<no value>");
```

### Containers and Spans

```cpp
template <class T>
void add_range(std::vector<T>& vec, dont_deduce<std::span<T const>> values)
{
    vec.resize(vec.size() + values.size());
    for (auto const& v : values)
        vec.push_back(v);
}

// otherwise this wouldn't work:
std::vector<float> vecA = ...;
std::vector<float> vecB = ...;
add_range(vecA, vecB);
```



## Summary

The (seemingly useless) `dont_deduce<T>` (or [std::type_identity](https://en.cppreference.com/w/cpp/types/type_identity)) can be used to selectively disable [template argument deduction](https://en.cppreference.com/w/cpp/language/template_argument_deduction).

This is a valuable tool in reducing API friction:

```cpp
template <class T>
vec3<T> operator+(vec3<T> const& a, T b); // (A)

template <class T>
vec3<T> operator+(vec3<T> const& a, dont_deduce<T> b); // (B)

vec3<float> v;
v + 3; // works with (B) but not with (A)
```

(_Title image from [pixabay](https://pixabay.com/photos/murder-the-scene-investigation-5294706/)_)
