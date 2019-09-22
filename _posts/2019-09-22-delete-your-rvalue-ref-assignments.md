---
layout: post
current: post
cover:  assets/images/rvalue-ref-assign.png
navigation: True
title: "Consider deleting your rvalue ref-qualified assignment operators"
date: 2019-09-22 8:00:00
tags: [c++, api design]
class: post-template
subclass: 'post'
author: philip
excerpt: 'Why is foo{} = foo{} working anyways?'
---

The title might sound like an incantation to summon some mid-tier C++ god but it addresses a very real everyday pitfall:

```cpp
struct foo { ... };
foo get_my_foo() { ... }

// .. some code later:
foo f;
get_my_foo() = f;
```

This [compiles](https://godbolt.org/z/oh3vFA) ... and does nothing useful.

We've assigned `f` to a temporary `foo`.
No error, no warning.

## A Real-Life Example

In the math library I'm writing we have a `mat` struct for matrices and `vec` for vectors.
Matrices are stored column-major, i.e. as an array of column vectors.
Now, sometimes you want to get the row of such matrix and thus `mat` has a function `vec mat::row(int)` that returns the specified row.
It has to return the `vec` per value because only columns are stored contiguously in `mat`:

```cpp
template <int C, int R, class ScalarT>
struct mat
{
    using col_t = vec<R, ScalarT>;
    using row_t = vec<C, ScalarT>;

    col_t columns[C]; // column-major matrix

    row_t row(int i) const { return ...; }
};

using mat3 = mat<3, 3, float>;
using vec3 = vec<3, float>;
```

And then someone writes:

```
mat3 m;
m.row(1) = {1, 2, 3};
```

Looks perfectly reasonable, compiles without warnings, ... and [does nothing](https://godbolt.org/z/azm1BP).


## Solution A

The problem in both cases is that it is totally fine to call `operator=` on an rvalue reference (`foo&&` or `vec3&&`).
The compiler-generated definition something looks like:

```cpp
struct foo
{
    foo& operator=(foo const&) = default;
    foo& operator=(foo&&) = default;
};
```

Note that these member functions are not `const`-qualified as assigning to a `const` object doesn't really make any sense.
An easy solution to our "easy to use accidentally wrong" API problem is thus:

```cpp
struct foo { ... };
const foo get_my_foo() { ... }

// .. some code later:
foo f;
get_my_foo() = f; // ERROR: cannot assign to const foo
```

## Solution B

Most of the time a type is used way more often than it is declared.
Our first solution requires each use of our type as a return value to be `const`-qualified.
Isn't there a solution that is write-once and then works any time `foo` is used as a return value?

Turns out there is.

And by the way, the following problem cannot be solved by `const`-qualification:

```cpp
foo f;
foo{} = f; // no error?
```

The `const`-solution works for return values but doesn't really prevent the core of the problem: 
assigning to a temporary.

Since C++11 it is possible to add [reference qualifiers to member functions](https://en.cppreference.com/w/cpp/language/member_functions#const-.2C_volatile-.2C_and_ref-qualified_member_functions).
These so-called ref-qualified member functions allow us to overload member functions not only on `const` and non-`const` but also on which type of reference `this` is (`&` for lvalue, `&&` for rvalue).

Simplifying a bit (a lot?), lvalues are "things with names", like local variables.
Most of the time, rvalues are temporaries or at least things we want to consider temporaries.

Thus, our second solution is to delete the assignment operators for rvalue ref-qualified `foo`s:

```cpp
struct foo
{
    foo& operator=(foo const&) & = default;
    foo& operator=(foo const&) && = delete;
    foo& operator=(foo&&) & = default;
    foo& operator=(foo&&) && = delete;
};
```

Unfortunately, declaring these [special member functions](https://foonathan.net/blog/2019/02/26/special-member-functions.html) causes the compiler-generated copy and move constructors to disappear.
That means for a full solution (see below) we also need to explicitly default copy and move ctors.
Declaring any constructor makes the implicitly declared default constructor disappear, so we need to declare that as well.

A minor consequence of deleting this assignment operator is that `std::is_assignable<foo, foo>` becomes `false`.
The reason is that `std::is_assignable<foo, foo>` is actually `std::is_assignable<foo&&, foo>`.
You typically only want `std::is_assignable<foo&, foo>` anyways, which is still `true`.


## Conclusion

If you suspect that a type of yours is susceptible to accidental assignment-to-temporary (like our `vec` is), consider deleting the rvalue ref-qualified assignments:

```cpp
struct foo
{
    // default ctor
    foo() = default;

    // copy and move ctor
    foo(foo const&) = default;
    foo(foo&&) = default;

    // assignment ops
    foo& operator=(foo const&) & = default;
    foo& operator=(foo const&) && = delete;
    foo& operator=(foo&&) & = default;
    foo& operator=(foo&&) && = delete;
};
```

So maybe the [~~rule-of-three~~ rule-of-five](https://en.wikipedia.org/wiki/Rule_of_three_(C%2B%2B_programming)) should now be rule-of-seven?
