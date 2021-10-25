---
layout: post
current: post
cover:  assets/images/move-van.jpg
navigation: True
title: "Moves in Returns"
date: 2021-10-23 02:00:00
tags: [c++, optimization]
class: post-template
subclass: 'post'
author: philip
excerpt: "Mini guide to 'when is my return a move?'"
---

Today we'll discuss code of the form:

```cpp
T work(/* ... */)
{
    /* ... */
    return x;
}
```

This is a classical "return-by-value" and (wrongfully) associated with copies and overhead.

In many cases, this will actually `move` the result instead of copying it.
For modern C++, one could even argue that this will move in _most_ cases (or, as we will see, completely _elide_ the copy and directly construct in the result memory).

This post discusses several common patterns and if they are moved, copies, or elided.

> Side note: _technically_ a move is a type of copy.
> For example, `T x = <expr>` performs a [copy initialization](https://en.cppreference.com/w/cpp/language/copy_initialization), which might select the move constructor during overload resolution.
> The rest of this post will use the colloquial "move" for "calls move ctor or assignment" and "copy" for "calls copy ctor or assignment".

# Tracking Construction and Assignment 

Reading the C++ standard (or cppreference) to reason about your code is valuable, but given the flood of information, it can be difficult to draw the correct inferences.
Thus, in addition to this _theoretical_ understanding, I love to validate my findings on [godbolt](https://godbolt.org/).

The following examples always use a type `T`, defined as:

```cpp
struct T
{
    T();                    // ctor
    ~T();                   // dtor
    T(T&&);                 // move ctor
    T(T const&);            // copy ctor
    T& operator=(T&&);      // move assignment
    T& operator=(T const&); // copy assignment
};
```

These functions are not implemented on purpose, so we will see the corresponding calls in the assembly.
Furthermore, I'll mark the `work` functions as `__attribute__((noinline))` so that we can see which special function calls belong where (caller or callee).

# Constructing Objects in Return

```cpp
T work()
{
    return T();
}

void use()
{
    auto obj = work();
}
```

Since C++17, this invokes **mandatory** [copy elision](https://en.cppreference.com/w/cpp/language/copy_elision).
No copy or move constructor is called, even if they have side effects.

[Assembly](https://godbolt.org/z/sYx6h8Mq7):

```cpp
work():
  ...
  call T::T() [complete object constructor]
  ...
  ret
use():
  ...
  call work()
  ...
  call T::~T() [complete object destructor]
  ...
  ret
```

`T` is constructed in `work` in a memory location that is provided by the caller `use`.
At the end of `use`, `T` is destructed.
No temporary copies are created, nothing is moved.

This "chains" in the sense that it also applies to `return other_work();` where `other_work` also returns a `T` by value.

With `return T();`, `work` will always call exactly one constructor and nothing else from `T`.
However, `use` is only this simple because we initialize `obj` with the result of `work()`.
If we assign it to an existing object, we get a temporary:

```cpp
T work()
{
    return T();
}

void use(T& obj)
{
    obj = work();
}
```

[Assembly](https://godbolt.org/z/Whcro3vno):
```cpp
work():
  ...
  call T::T() [complete object constructor]
  ...
  ret
use(T&):
  ...
  call work()
  ...
  call T::operator=(T&&)
  ...
  call T::~T() [complete object destructor]
  ...
  ret
```

`work` constructs a `T` for which `use` provides the stack space.
This temporary `T` is then _moved_ into `obj` using `T::operator=(T&&)`.
Finally, the temporary `T` is destroyed.

Before C++17, this type of optimization was allowed, but optional.
In particular, if your object is neither copyable nor movable, you could run into compile errors depending on if this optimization was applied or not (e.g. debug vs. release).

> Note: In C++17, this direct creation of the result in space provided by the caller has the fancy name "unmaterialized value passing".

# Returning a Local Variable

```cpp
T work()
{
    T obj;
    // ...
    return obj;
}

void use()
{
    auto obj = work();
}
```

Interestingly enough, this results in the same [assembly](https://godbolt.org/z/nq6zTPKEo) as our previous case:

```cpp
work():
  ...
  call T::T() [complete object constructor]
  ...
  ret
use():
  ...
  call work()
  ...
  call T::~T() [complete object destructor]
  ...
  ret
```

No temporary object is created, nothing is moved or copied.
However, this form of [copy elision](https://en.cppreference.com/w/cpp/language/copy_elision) is not mandatory.
This is also known as "named return value optimization" or NRVO.
Note that in this case, in contrast to the previous case, `T` must be copyable or movable, even if the actual copy or move constructor is not called in the end.

It gets more interesting if we have other ways out of the function:

```cpp
T work()
{
    if (some_condition())
        return T();

    T obj;
    return obj;
}
```

[Assembly](https://godbolt.org/z/o7Ws33fj1):

```cpp
work():
  ...
  call some_condition()
  if:
    call T::T() [complete object constructor]
    ...
    ret
  else:
    ...
    call T::T() [complete object constructor]
    ...
    call T::T(T&&) [complete object constructor]
    ...
    call T::~T() [complete object destructor]
    ...
    ret
```

If the condition is `true`, we construct the result directly as before.
However, the second case now creates a temporary `T`.
This temporary is then move-constructed into the return value.
Afterwards, the temporary is destructed.

No copy elision was performed for `obj` (though I am not 100% sure why. It should be allowed and possible here.)
Still, the result is a _move_, not a _copy_.
This is a feature of the [return statement](https://en.cppreference.com/w/cpp/language/return):
Since C++11, `return x` (or `return (x)` or `return ((x))` for that matter) will try to use the move constructor if `x` is a local variable or a function parameter.

> Note: the actual rule has a few nuances, but this is a good first-order approximation.

# Moving from a Local Variable

You might have seen the following:

```cpp
T work()
{
    T obj;
    return std::move(obj);
}
```

Many compilers, IDEs, and linters warn about this.
GCC might say "moving a local object in a return statement prevents copy elision".
And indeed, the [assembly](https://godbolt.org/z/q6cbzWzrx) is now:

```cpp
work():
  ...
  call T::T() [complete object constructor]
  ...
  call T::T(T&&) [complete object constructor]
  ...
  call T::~T() [complete object destructor]
  ...
  ret
```

A temporary that is move-constructed into the return value.
Without the `std::move`, we had no move construction at all.

# Returning a Function Parameter

Local variables and function parameters have slightly different behavior.

```cpp
T work(T obj)
{
    return obj;
}

void use()
{
    auto obj = work(T());
}
```

When `obj` was a local variable, we had the freedom to change _where_ it is allocated.
If all paths through the function end in `return obj`, the compiler could use the caller-provided space for the return value, thus _eliding_ any move into the result.

However, function parameters are already allocated by the caller and distinct from the return value.
Luckily, the rules for `return` statements still apply and we get a move in the [assembly](https://godbolt.org/z/xj3Wqo8Wb):

```cpp
work(T):
  ...
  call T::T(T&&) [complete object constructor]
  ...
  ret
use():
  ...
  call T::T() [complete object constructor]
  ...
  call work(T)
  ...
  call T::~T() [complete object destructor]
  ...
  call T::~T() [complete object destructor]
  ...
  ret
```

Even passed-by-value, no `T` is copied in this whole example.
The caller (`use`) creates a `T` where `work` will expect it.
`work` itself only move-constructs `T` in the return value.
`use` then destructs the argument `T` ("at the end of the statement"), followed by destruction of the result of `work` ("at the end of the scope").

# Non-Matching Types

Copy elision only works if the result type matches what we want to return.
This might not always be the case.
Something I find myself writing with decent frequency:

```cpp
std::optional<T> work()
{
    if (some_condition())
        return std::nullopt;

    return T();
}
```

And the obvious question is if the second `return` _copies_ or _moves_ the `T` into the `optional<T>`.

More general, let's say we have a second type `U` and `T` has implicit conversions:

```cpp
struct T
{
    ...
    T(U&&) noexcept;      // "move-convert"
    T(U const&) noexcept; // "copy-convert"
};
```

Now we can ask the question what the following code will call:

```cpp
T work()
{
    return U();
}
```

or

```cpp
T work()
{
    U obj;
    return obj;
}
```

Both result in the [same](https://godbolt.org/z/cW5f1K65b) [assembly](https://godbolt.org/z/5jsbcWeKz):

```cpp
work():
  ...
  call U::U() [complete object constructor]
  ...
  call T::T(U&&) [complete object constructor]
  ...
  call U::~U() [complete object destructor]
  ...
  ret
```

A temporary `U` is created, move-"converted" into the result `T`, and then destructed.
No copy involved.

> Fun fact: in "vanilla" C++11, this created a copy.
> The behavior was fixed in C++14 and back-ported via [defect report](https://wg21.cmeerw.net/cwg/issue1579).
> Thus, most compiler with C++14 support will emit the move, even if you explicitly compile for C++11.
> However, pre-C++14 compiler might emit the copy.


# Where Copy??

It delights me to see so many cases where the default (without any `std::move` involved) will result in either move construction or even complete elision of copy or move.

So, when will return-by-value actually copy?
A small collection of patterns to look out for:

```cpp
T work()
{
    struct { T t; } v;
    return v.t; // COPY! returning a member
}

T work()
{
    static T globalT;
    return globalT; // COPY! not a local var
}

T work(T& obj)
{
    return obj; // COPY! T& matches T const&, not T&& 
}

T work(T const& obj)
{
    return obj; // COPY!
}

T work(T&& obj)
{
    return obj; // COPY! "inside" work, obj is an lvalue
                // NOTE: careful with lifetimes here
                // NOTE: is a move in C++20
}

T work(T const obj)
{
    return obj; // COPY! T const cannot be moved
                // NOTE: const is ignored for the signature,
                //       which is work(T) and not work(T const)
                //       but has "effect" inside the function
}
```

Maybe unexpectedly, the following [is elided](https://godbolt.org/z/heveTn3EW):

```cpp
T work()
{
    T const obj;
    return obj; // no copy or move involved
}
```

Because while we cannot move `T const`, we can still allocate it directly "in" the return value.
Still, this is a bit brittle, as a slightly more complex function will create a copy:

```cpp
T work()
{
    if (some_condition())
        return T();

    T const obj;
    return obj; // now it's a copy
}

T work()
{
    T const obj;
    return std::move(obj); // also a copy
                           // T const&& matches T const&, not T&&
}
```

# Conclusion

We saw many examples where modern C++ now naturally creates moves instead of copies or even elides them altogether and directly constructs the return value "in the proper location".
As the last section showed, you still have to look out for references or non-local variables.
This is probably a good thing, because those tend to have multiple aliases, which might take offense if their data suddenly moved away.

Most of the explanations in this post are somewhat simplified to make them palatable.
In particular, exceptions and `volatile` variables can complicate the situation a lot.
Also, keep in mind that inside a lambda, captures are not considered local variables.

The assembly shown in the examples can be considered a kind of worst case scenario, as the compiler has no access to the special member functions of `T`.
When these functions are visible, they can often be inlined and further optimized.


(_Title image from [unsplash](https://unsplash.com/photos/3vlGNkDep4E)_)
