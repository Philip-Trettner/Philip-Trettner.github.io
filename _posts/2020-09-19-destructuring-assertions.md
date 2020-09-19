---
layout: post
current: post
cover:  assets/images/blueprint.jpg
navigation: True
title: Destructuring Assertions
date: 2020-09-19 02:00:00
tags: [c++]
class: post-template
subclass: 'post'
author: philip
excerpt: 'How can we make ASSERT(a == b) print the values of a and b?'
---

Assertions are a major tool in defensive programming and I consider it a symbol of a mature programmer when their code is liberally accompanied by assertions.
They embody a fail-fast mentality and serve as additional documentation, making many assumptions explicit that the programmer made during the implementation.

```cpp
#include <cassert>

auto a = 1;
auto b = 2;
assert(a == b);
```

In this post we will remedy a shortcoming of the traditional C assertion:

```
Assertion `a == b' failed.
```

Okay, our assertion failed, our code (or assumption) is buggy. But what are `a` and `b`?

Test frameworks like [Catch2](https://github.com/catchorg/Catch2) or [doctest](https://github.com/onqtam/doctest) are (seemingly magically) able to display the values of `a` and `b` when their assertions / checking macros fail:

```
Example.cpp:7: FAILED:
  REQUIRE( a == b )
with expansion:
  1 == 2
```

The rest of this post explains how to display the values of `a` and `b`.

> SPOILER: we're going to exploit operator precedence and break some macro hygiene. `a == b` will be expanded to `assert_t{} < a == b`, which is then parsed as `(assert_t{} < a) == b`, allowing access to `a` and `b`. 


## Typical Assertion Anatomy

Before we start destructuring the assertion expression, let's take a look at how assertions are typically implemented:

```cpp
void on_assert_failed(char const* expr, char const* file, int line, char const* fun);

#define ASSERT(expr) (static_cast<bool>(expr) ?                                  \
                      void(0) :                                                  \
                      on_assert_failed(#expr, __FILE__, __LINE__, __FUNCTION__))
```

There is already something noteworthy going on here.
A super naive version would be:

```cpp
#define ASSERT(expr) if (!expr) \
                         on_assert_failed(#expr, __FILE__, __LINE__, __FUNCTION__);
```

Most of this is basic macro hygiene but it cannot hurt to repeat it.

`!expr` is dangerous as `ASSERT(a == b)` would expand to `if (!a == b)`.
The common fix of `!(expr)` is slightly better but still dangerous as the additional parentheses silence warnings, e.g. for the typo in `ASSERT(a = b)`.
A better solution is `!static_cast<bool>(expr)` which preserves most warnings.

Secondly, in function-like macros, it is common courtesy to make them behave as if they were normal functions.
On the one hand, this means requiring a semicolon at the end.
On the other hand, it means that one should implement `ASSERT` as an expression, not a statement.

```cpp
// should error due to missing ;
ASSERT(a == b) 

// should work as expected
if (some_condition)
    ASSERT(a == b);
else
    ASSERT(a != b);
```

Note that the `else` would attach to the `if (!expr)` of the naive version, NOT the expected `if (some_condition)`.
This is the reason why expressions are preferred.
For the assertion the ternary operator `cond ? true_expr : false_expr` is sufficient.
If you want to execute multiple statements, the `do { ... statements ... } while(0)` construct is popular.
It is not an expression but at least it interacts properly with other control flow structures.


## Destructuring Simple Expressions

So, now that we know how a basic assertion works, how do we "analyze" the asserted expression to get `a` and `b` in `ASSERT(a == b)`?
The metaprogramming capabilities of C++ do not allow us to inspect arbitrary expression as for example [Nim Macros](https://hookrace.net/blog/introduction-to-metaprogramming-in-nim/#macros) are able to. 
What can we do instead?

If `ASSERT` were a normal function, `a == b` would be evaluated before calling the function and there would be no chance to get the values of `a` and `b`.
However, we are in a macro setting where the expression is "embedded" into the macro body via token substitution.
While we cannot change `a == b`, we can control its surroundings.

How does this help us?

Our goal is to "snatch" `a` from `a == b`, store its value AND string representation, then compare against `b`, while also storing `b`s string representation.
If the comparison fails, we call the "assertion failed" handler while passing the representation of `a` and `b`.

As already spoilered, we will exploit [operator precedence](https://en.cppreference.com/w/cpp/language/operator_precedence).

We are going to surround `a == b` with `assert_t{} OP a == b` where `assert_t` is a helper type and `OP` is our "snatching" operator.
Comparisons are associated left-to-right, so `OP` must have the same or higher precedence than the comparisons `==, !=, <, <=, >, >=`.
However, when its precedence is too high, it will interface with more complex assertions such as `ASSERT(a + b == c)` where we want to "snatch" `a + b` and not only `a`.
Looking at the precedence table, this leaves us with the shift operators or `<, <=, >, >=` (ignoring the C++20 `<=>` spaceship).

For no particular reason I'll continue with `<`:

```cpp
void set_assert_vars(std::string_view a, std::string_view b, std::string_view comp);
void on_assert_failed(char const* expr, char const* file, int line, char const* fun);

#define ASSERT(expr) ((assert_t{} < expr) ?                                       \
                       void(0) :                                                  \
                       on_assert_failed(#expr, __FILE__, __LINE__, __FUNCTION__))

template <class A>
struct check_t
{
    A a;

    template <class B>
    bool operator==(B&& b) const
    {
        if (a == b)
            return true;

        set_assert_vars(std::to_string(a), std::to_string(b), "==");
        return false;
    }
};

struct assert_t
{
    template <class A>
    check_t<A> operator<(A&& a)
    {
        // prevent copies of a
        return check_t<A>{std::forward<A>(a)};
    }
};
```

Consider for example `ASSERT(1 + 1 == 3);`.
Inside the macro, this expands to the condition `assert_t{} < 1 + 1 == 3`, which is parsed as `(assert_t{} < 1 + 1) == 3`.
This calls `operator<` of `assert_t`, returning a `check_t<int>` with member `a` set to `2`.
`check_t` in turn has an `operator==` that is called with `3` as its right-hand side.
The comparison `if (a == b)` fails, at which point `set_assert_vars` is called.
Only now are `a` and `b` converted to strings.
This is important because `to_string` is kinda expensive and we don't want to slow down runtime performance when the assertion is not failing.
We know that the "assertion failed" handler will be called immediately afterwards, so `set_assert_vars` can simply store its arguments in `thread_local` global variables that the handler will then display.

See [here](https://godbolt.org/z/MsGvM3) for a fully working example.

```cpp
ASSERT(1 + 1 == 3);
```

```
assertion '1 + 1 == 3' failed
  in ./example.cpp:55 (main)
  expansion: 2 == 3
```


## Next Steps

To make this production-ready I would recommend the following:

* add all desired comparison operators to `check_t`
* add `static_assert`s to `check_t` that check if the comparison between `A` and `B` actually works (nicer compile errors)
* move `assert_t` and `check_t` in "`detail::`" or "`impl::`" scopes
* write a user-extensible version of `std::to_string` so that user types can register their own formatter
* try to remove the dependence on `<string>` as this is a rather expensive header (for example, the custom `to_string` might return a `char const*` that was allocated via `new` and is `delete[]`d by `on_assert_failed`; this is not performance critical)
* add a `operator bool()` to `check_t` to support `ASSERT(some_bool)`
* add `operator&&` and `operator||` to `check_t` and `assert_t` that cause `static_assert` failures (we cannot destructure chained expressions so this should be forbidden. there is an escape hatch via `ASSERT((a || b))` without destructuring)
* only store a reference in `check_t` so that types must not even be movable (lifetime is fine as the reference doesn't outlive the assert expression)
* add an optional general message to the assertion, supporting a format-like syntax (e.g. `ASSERTF(a == f(b), "xyz is not fulfilled and b is {}", b);`)
* proper integration with logging, stack traces, custom assert handlers


## Summary

While the metaprogramming capabilities of C++ are not expressive enough to analyze expression ASTs, we nevertheless can achieve simple "destructuring" of comparisons to implement assertions that report the compared values:

```cpp
auto a = 1;
auto b = 2;
auto c = 2;
auto d = 3;

ASSERT(a + b == c + d);

// assertion 'a + b == c + d' failed
//   in ./example.cpp:55 (main)
//   expansion: 3 == 5
```

This works by breaking macro hygiene and use operator precedence to "snatch" the compared value before it is actually compared.

```cpp
ASSERT(a + b == c + d);

// is expanded to
((assert_t{} < a + b == c + d) ? void(0) :
                                 on_assert_failed(...));

// is parsed as
(((assert_t{} < a + b) == c + d) ? void(0) :
                                   on_assert_failed(...));
```
