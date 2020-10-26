---
layout: post
current: post
cover:  assets/images/register.jpg
navigation: True
title: "Static Registration Macros"
date: 2020-10-17 04:00:00
tags: [c++]
class: post-template
subclass: 'post'
author: philip
excerpt: "Central registration of types or functions just doesn't feel very DRY sometimes."
---

```cpp
TEST("my test case")
{
    auto a = 1 + 1;
    auto b = 2;
    CHECK(a == b);
}
```

You might have seen similar code in a testing framework [of](https://github.com/google/googletest) [your](https://github.com/catchorg/Catch2) [choice](https://github.com/onqtam/doctest).
This example even sports a nifty [destructuring assertion](/blog/2020/09/19/destructuring-assertions) that will print the values of `a` and `b` should the check fail, though that is not the focus today.

One question that arises from time to time for code like this: how is this code even executed when it doesn't contain a `main` function and is never referenced elsewhere?
Worse, you can put that in a source file that has zero overlap with your other translation units, potentially not even a shared header, and it works.

So in today's episode of How It's Made, we will construct such a _static registration macro_ from scratch.
This will touch many intermediate C++ concepts that might be trivial to some, but worth repeating to others.

The end result is a macro that enables decentralized static registration of functions or types and can be used to reduce code duplication and unnecessary file coupling.
It can also reduce errors by keeping definition and registration code close to each other.
Of course, one drawback is that there is no central location where you can see all registrations.
Thus, as always, it is the responsibility of the developer to decide if this technique represents the local optimum in the particular trade-off space at hand.
In my opinion, it is a worthwhile tool to have at one's disposal.

While creating this tool, we will (re)visit the following topics:

* how to execute code before main
* what is [static storage duration](https://en.cppreference.com/w/cpp/language/storage_duration#Storage_duration)?
* evading the [static initialization order fiasco](https://en.cppreference.com/w/cpp/language/siof) using the _construct on first use_ idiom
* using `static` and [unnamed namespaces](https://en.cppreference.com/w/cpp/language/namespace#Unnamed_namespaces) to prevent [ODR](https://en.cppreference.com/w/cpp/language/definition) violations and linker goulash
* use `__LINE__` to support multiple registrations per file
* writing a macro that concatenates two identifiers that might contain further macros
* adding parameters to the registered function
* approximating extensible named arguments
* a minimal boilerplate macro-free version

This post is deliberately a little meandering, briefly explaining relevant C++ concepts during each step.
If you are just interested in the result, you can skip directly to the [final version](#final-version).


## Most Basic Version

Let's start with the foundation:
How can one write code that is executed without being referenced?

The answer is surprisingly simple:

```cpp
struct foo
{
    foo() { std::cout << "look ma, before main!" << std::endl; }
};

// outside of a function:
foo f;
```

Here, `f` is declared at namespace level, e.g. the global namespace, and thus has [static storage duration](https://en.cppreference.com/w/cpp/language/storage_duration#Storage_duration).
Objects with static storage duration are allocated (and their constructor is called) _before_ the first statement of your `main()` function.
Consequently, they are destroyed when the program ends.
(Initialization in C++ is, of course, a mess, so [some exceptions apply](https://en.cppreference.com/w/cpp/language/initialization#Non-local_variables))

Thus, if we want to register code automatically, before `main`, we might be tempted to write:

```cpp
using fun_t = void(*)();
std::vector<fun_t> registered_functions;

void my_fun()
{
    // user-code ...
}

struct foo
{
    foo() { registered_functions.push_back(my_fun); }
};

foo f;

int main()
{
    for (auto f : registered_functions)
        f();
}
```

> Did you know? The [`main` function](https://en.cppreference.com/w/cpp/language/main_function) in C++ is the only non-void function that has defined behavior when you don't return: an implicit `return 0`.


## Initialization Order

We now want to "scale up" our previous code and use our technique among multiple files.
Thus, we write:

```cpp
// A.hh
#pragma once

using fun_t = void(*)();
void register_function(fun_t f);
void execute_registered_functions();


// A.cc
#include <A.hh>

std::vector<fun_t> registered_functions;

void register_function(fun_t f) 
{ 
    registered_functions.push_back(f); 
}

void execute_registered_functions()
{
    for (auto f : registered_functions)
        f();
}


// B.cc
#include <A.hh>

void my_fun()
{
    // user-code ...
}

struct foo
{
    foo() { register_function(my_fun); }
};

foo f;


// main.cc
#include <A.hh>

int main()
{
    execute_registered_functions();
}
```

Congratulations! 
We are now a victim of the [static initialization order fiasco](https://en.cppreference.com/w/cpp/language/siof).
Objects with static storage duration are initialized before `main()`, but their relative order is not specified.
Inside the same translation unit, it is top-to-bottom.
Outside, it might depend on the order in which the files are passed to the compiler (... or the current day of the week).

In our case, `foo f` in `B.cc` might get initialized before `registered_functions` in `A.cc`, thus either crash on start (as `registered_functions` could contain uninitialized memory), ignore the registration (if the default ctor of `registered_functions` "clears" the vector), or even occasionally work (if a zero-initialized vector is valid and its default ctor does nothing).
Accessing `registered_functions` before it's initialized is undefined behavior but in practice, your code might sometimes work, sometimes not, sometimes crash.

An effective solution is the so called _construct on first use_ idiom, similar to how singletons are often implemented.

My preferred implementation of this idiom are function-local `static` variables.
These also have static storage duration, but [are initialized when the declaration "is executed"](https://en.cppreference.com/w/cpp/language/storage_duration#Static_local_variables).
While not relevant in our case, note that function-local `static` initialization is guaranteed to be thread-safe since C++11.

Okay, let's fix the initialization order problem:

```cpp
// A.cc
#include <A.hh>

std::vector<fun_t>& registered_functions()
{
    static std::vector<fun_t> v;
    return v;
}

void register_function(fun_t f) 
{ 
    registered_functions().push_back(f); 
}

void execute_registered_functions()
{
    for (auto f : registered_functions())
        f();
}
```

Now it doesn't matter in which order the translation units are initialized, `registered_functions()` will always construct the `static std::vector<fun_t> v` on the first call.

## ODR Violations

While we fixed the initialization order problem, we still violate one of C++'s most (in)famous rules: the [One Definition Rule](https://en.cppreference.com/w/cpp/language/definition#One_Definition_Rule)

> Only one definition of any variable, function, class type, enumeration type, concept, or template is allowed in any one translation unit (some of these may have multiple declarations, but only one definition is allowed).

The most notable exception are functions that are either [explicitly or implicitly defined `inline`](https://en.cppreference.com/w/cpp/language/inline).
Implicit `inline` is surprisingly common: functions that are directly defined inside a class/struct/union, `constexpr` functions, fully defined templated functions, `constexpr` static data members.

> Opinion: I consider `inline` one of the most confusing keywords for newcomers, though `static` comes close.
> I keep being surprised how many people (even those that have used C++ for years) still believe `inline` is for [inlining functions](https://en.wikipedia.org/wiki/Inline_expansion).
> While it might have been the original intent and [some modern compilers still take it as an optimization hint](https://blog.tartanllama.xyz/inline-hints/), it is mostly misleading.
> `inline` is mainly about telling the linker to shut up about multiple definitions and pinky-promising that all definitions are exactly the same.
> Secondary purpose is to guarantee that function pointers and inline variable addresses are the same across translation units.

So, how did we violate the ODR?

Not yet, but we are almost begging for it.
Consider what happens if we add a new file:

```cpp
// C.cc
#include <A.hh>

void my_fun()
{
    // more user-code ...
}

struct foo
{
    foo() { register_function(my_fun); }
};

foo f;
```

`foo`, `f`, and `my_fun` exist in `B.cc` and `C.cc` and do different things.
But neither "sees" the other version, so this will most likely compile.
ODR is [no diagnostic required](https://en.cppreference.com/w/cpp/language/ndr) and in my experience, it is often not diagnosed.
Modern linkers became better diagnosing this kind of problem, though not with 100% accuracy.
In this case, the [ld.gold linker](https://en.wikipedia.org/wiki/Gold_(linker)) actually complains:

```cpp
/usr/bin/ld.gold: error: B.cc.o: multiple definition of 'my_fun()'
/usr/bin/ld.gold: C.cc.o: previous definition here
```

(Note that the _compiler_ is often unable to diagnose this, but the _linker_ can.)

The problem here is that `f` and `my_fun` have [external linkage](https://en.cppreference.com/w/cpp/language/storage_duration#external_linkage).
They are available to other translation units and are exported as symbols that are resolved by the linker.

However, we never intended those to be visible to other TUs.
They are only a vehicle to implement our static registration.
Even `my_fun` should not be visible, we just want to access the function pointer later.

The solution is to use [internal linkage](https://en.cppreference.com/w/cpp/language/storage_duration#internal_linkage).
Names with internal linkage are "local" to a translation unit and are neither exported as symbols nor do they conflict with identical names from other translation units.

There are two main mechanisms to switch to internal linkage:
`static` functions or variables, and [unnamed namespaces](https://en.cppreference.com/w/cpp/language/namespace#Unnamed_namespaces), also known as anonymous namespaces.

Note that `foo::foo()` is defined inside `struct foo` and thus is implicitly `inline`.
However, it still has external linkage and will conflict with `foo`s defined in other TUs.
Worse, because they are `inline`, you will not get a `multiple definition` error but the linker will arbitrarily pick one definition, almost always leading to weird errors.

Thus, we arrive at the first, safely usable version:

```cpp
// A.cc
...
static std::vector<fun_t>& registered_functions()
{
    static std::vector<fun_t> v;
    return v;
}
...


// B.cc / C.cc
#include <A.hh>

namespace
{
void my_fun()
{
    // more user-code ...
}

struct foo
{
    foo() { register_function(my_fun); }
};

foo f;
}
```


## Reducing Boilerplate via Macro

Macros don't have the best reputation in C++ as they are [rather unhygienic](https://en.wikipedia.org/wiki/Hygienic_macro), lead to [double expansion errors](https://gcc.gnu.org/onlinedocs/cpp/Duplication-of-Side-Effects.html), [don't respect namespaces](http://www.suodenjoki.dk/us/archive/2010/min-max.htm), interact poorly with IDE features such as renaming, among others.

Still, they are sometimes the local optimum for reducing boilerplate and [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) violations.

In this case, I'd argue that a macro is justified to hide the noisy implementation detail:

```cpp
#define REGISTER(Name)                                   \
    static_assert(true, Name " must be string literal"); \
    static void my_fun();                                \
    namespace                                            \
    {                                                    \
    struct foo                                           \
    {                                                    \
        foo() { register_function(Name, my_fun); }       \
    } f;                                                 \
    }                                                    \
    static void my_fun()
```

which can then be used as:

```cpp
REGISTER("my fun")
{
    // user code
}
```

Apart from introducing a macro, I made some slight adjustments.

Firstly, we often want to associate a name with whatever we registered, so I assumed that we can now register via `void register_function(char const* name, fun_t f)`.
The `static_assert(true, Name " must be string literal");` is an optional safeguard to guarantee that only string literals are used in the `REGISTER` macro:

```cpp
<source>:8:5: error: expected string literal for diagnostic message in static_assert
REGISTER(1)
         ^
```

Its use is optional, some people prefer to use `register_function(#Name, my_fun);` to stringify `Name` or even extend it to `REGISTER(...)` and `#__VA_ARGS__`.

Secondly, for brevity, we can write `struct foo { /* ... */ } f;` to define a `struct foo` and a variable `foo f` at the same time.

And finally, by forward declaring `my_fun` (which must be visible in `foo::foo()`) and defining it at the end without `{ /* ... */ }`, we enable the quite intuitive and readable `REGISTER("my fun") { /* ... */ }` syntax.


## Increasing Macro Hygiene

While we have cleaned up the registration code, it still introduces (TU-local) identifiers that can easily conflict with other use code (`my_fun`, `f`, `foo` are not _that_ exotic).
Behind the macro, their definitions are now basically invisible.
Though unlikely to introduce silent errors, it can still lead to unexpected and arbitrary-seeming compile errors.

Worse, we can currently only register a single function per TU.
Trying to use `REGISTER("some name")` twice, even with different names, leads to duplicated definitions for `foo`, `f`, and `my_fun`.

We start by choosing identifiers that are less likely to conflict.
We don't have to fully uglify our code like the [standard library has to](https://www.reddit.com/r/cpp/comments/h0flxv/why_is_std_implementation_so_damn_ugly/).
Even if we wanted to, [we are technically not allowed to](https://en.cppreference.com/w/cpp/language/identifiers#In_declarations).

However, this will not solve the multiple registrations per file problem.
For that we need unique identifiers per file.
Using `a##b`, we can concatenate identifiers in macros.
While we cannot use `Name` (a string literal), we can use `__LINE__`, the current line number, to create our unique names.
That will allow us to register any number of functions per file, as long as we don't register two functions on the same line.

In our naivety, we write:

```cpp
#define REGISTER(Name) \
  ...
  static void my_fun_##__LINE__();
  ...
```

... which [does not work](https://godbolt.org/z/8W8vch):

```cpp
<source>:19:1: error: redefinition of 'foo__LINE__'
REGISTER("fun b")
^
<source>:8:12: note: expanded from macro 'REGISTER'
    struct foo##__LINE__
           ^
```

Turns out, `a##b` does not expand `a` or `b` if they are macros themselves.

> A ## operator between any two successive identifiers in the replacement-list runs parameter replacement on the two identifiers **(which are not macro-expanded first)** and then concatenates the result. This operation is called "concatenation" or "token pasting".

Thus, we need the popular "two-step" macro concatenation:

```cpp
#define CONCAT_IMPL(a, b) a##b
#define CONCAT(a, b) CONCAT_IMPL(a, b)
```

The details are [somewhat](https://gcc.gnu.org/onlinedocs/cpp/Macro-Arguments.html) [arcane](https://gcc.gnu.org/onlinedocs/cpp/Argument-Prescan.html).
The gist is: if you want to expand macros that in a concatenation, you need to go "one level deeper":
calling `CONCAT_IMPL(my_fun, __LINE__)` creates `my_fun__LINE__`, but `CONCAT(my_fun, __LINE__)` creates `CONCAT_IMPL(my_fun, __LINE__)`, then does the so called [argument prescan](https://gcc.gnu.org/onlinedocs/cpp/Argument-Prescan.html) which does a complete expansion to `CONCAT_IMPL(my_fun, 17)` and finally `my_fun17` (assuming we call the macro on line 17).

For our hygienic version, I've decided to put all declarations into a `detail::` namespace and additionally choose long-ish names starting with an underscore (which is allowed as long as it's not the global namespace and the second character is neither underscore nor capital):

```cpp
#define REGISTER(Name)                                                   \
    static_assert(true, Name " must be string literal");                 \
    namespace detail                                                     \
    {                                                                    \
    /* function we later define */                                       \
    static void CONCAT(_registered_fun_, __LINE__)();                    \
                                                                         \
    namespace /* ensure internal linkage for struct */                   \
    {                                                                    \
    /* helper struct for static registration in ctor */                  \
    struct CONCAT(_register_struct_, __LINE__)                           \
    {                                                                    \
        CONCAT(_register_struct_, __LINE__)()                            \
        { /* called once before main */                                  \
            register_function(Name, CONCAT(_registered_fun_, __LINE__)); \
        }                                                                \
    } CONCAT(_register_struct_instance_, __LINE__);                      \
    }                                                                    \
    }                                                                    \
    /* now actually defined to allow REGISTER("name") { ... } syntax */  \
    void detail::CONCAT(_registered_fun_, __LINE__)()
```

With this, we can [register multiple functions per TU](https://godbolt.org/z/cb36d7).
Note that `_registered_fun_` must be static and cannot be inside the unnamed namespace as [our out-of-line definition would not work otherwise](https://godbolt.org/z/19aKPW).

Finally, instead of `__LINE__`, it is possible to use `__COUNTER__` instead, which is supported by the major compilers.
However, it does not work by simply replacing `__LINE__` by it as it would generate different names for the different `CONCAT(_registered_fun_, __COUNTER__)` instances inside our macro.
A solution would be to create a helper `REGISTER_IMPL(Name, ID)` macro and `#define REGISTER(Name) REGISTER_IMPL(Name, __COUNTER__)`.
I usually don't need to register more than one function per line and go with the `__LINE__` version.

In production, it is often useful to add `__LINE__` and `__FILE__` to `register_function`.
For example, this can be used to output where a test was defined when an assertion failed.


## Adding Parameters

We now have a basic, hygienic static registration macro.
This is often already enough for the desired purposes (e.g. declaring tests or registering polymorphic types in a deserialization system).

However, sometimes the registered functions themselves have parameters.
For example, in my own testing library, one can declare fuzz tests via:

```cpp
FUZZ_TEST("my test")(tg::rng& rng)
{
    float a = uniform(rng, -10.f, 10.f);
    float b = uniform(rng, -10.f, 10.f);
    CHECK((a + b) * (a + b) <= a * a + b * b);
}
```

Here, `tg::rng&` is a pseudorandom number generator provided by the testing library.
The fuzz tests should be deterministic relative to the `rng` which makes it possible to exactly reproduce failing random tests by providing the same seed again.

Of course, this can be simply added to the macro by either hard-coding the parameters in both declaration and definition, or by `#define REGISTER(Name, ...)` and use `__VA_ARGS__` in declaration and definition:

```cpp
// solution 1: (tg::rng& rng) is hard-coded in REGISTER(Name)
REGISTER("my test")
{
    auto a = uniform(rng, -10.f, 10.f);
    CHECK(std::abs(a) <= 10);
}

// solution 2: __VA_ARGS__ is used to change declaration and definition
REGISTER("my test", tg::rng& rng)
{
    auto a = uniform(rng, -10.f, 10.f);
    CHECK(std::abs(a) <= 10);
}
```

I dislike solution 1 because `rng` becomes invisible at use-site.
Readers have to know that it's available and how it's called.

The second solution works fine and modern IDEs often even provide decent support for refactoring (e.g. renaming `rng`), though it is far from guaranteed.

This can also be used to support different signatures with the same macro:

```cpp
template <class F>
void register_function(char const* name, F&& f);
{
    if constexpr (std::is_invocable_v<F, int>)
        register_int_version(name, f);
    else if constexpr (std::is_invocable_v<F, float>)
        register_float_version(name, f);
    else
        static_assert(always_false<F>, "only int and float versions are supported");
}
```

(Where `always_false<T>` is [the helper I've blogged about before](/blog/2020/10/03/always-false).)

While the second solution is definitely not bad, I like to use the variadic macro for named options instead (see next section).
For the `FUZZ_TEST` macro, I only hard-coded the `static void CONCAT(_registered_fun_, __LINE__)(tg::rng&);` declaration, so the user has to provide the parameters for the definition _outside_ the macro.
This allows choosing a different name for the parameter but NOT a different type, which may or may not be desirable.

If the parameters should be fully user-defined (e.g. with a templated or overloaded `register_function`) **and** the parameters should be outside the macro invocation, then the only way I could think of uses lambdas and requires an `;` after the closing `}`.
The idea is:

```cpp
namespace
{
struct foo
{
    template <class F>
    foo(F&& f)
    {
        register_function(Name, f);
    }
};
}
static foo f = [] /* end of macro */
```

which can then be used as:

```cpp
REGISTER("my fun")(int a, int b)
{
    // user code
};
```

(Unfortunately, clang format butchers the formatting.)

> If you find a way to have user-specified parameters _outside_ the macro and _without_ a closing `;`, please let me know!


## Extensible Named-Argument Approximation

Continuing with my test framework example, sometimes we want to disable a test, start it with a specific seed, or always run it after some other test.
With a variadic register macro, we can realize the following:

```cpp
TEST("test A") { ... }

FUZZ_TEST("test B", after("test A"), seed(123456))(tg::rng& rng) { ... }

TEST("test C", disabled) { ... }
```

> Note: I quite like this syntax, though I accept that it is a bit "too much syntactic sugar" for some people.
> It's discoverability is not as high as for normal member functions but it is low-noise, flexible, and extensible.

Let's say our test framework has the namespace `tf`.
We define the following:

```cpp
namespace tf::config
{
struct after
{
    explicit after(char const* p) : pattern(p) { }
    char const* pattern;
};

static constexpr struct disabled_t
{
} disabled;

struct seed
{
    explicit seed(size_t v) : value(v) { }
    size_t value;
};
}
```

This is our extensible config namespace.
Instead of a `register_function`, we have:

```cpp
tf::Test& register_test(char const* name, fun_t f);
```

This allocates a `Test` object and returns a reference to it.
We add:

```cpp
namespace tf
{
    void configure(Test& test, config::after const&);
    void configure(Test& test, config::disabled_t const&);
    void configure(Test& test, config::seed const&);

    template <class... Args>
    void do_configure(Test& test, std::string_view name, Args&&... args)
    {
        test.setName(name);
        (configure(test, std::forward<Args>(args)), ...);
    }
}
```

Where each `configure` sets / changes appropriate members in `Test`.
`do_configure` is a variadic helper that sets the test name and calls `configure` for each (perfectly forwarded) argument.
The final piece is our variadic macro `REGISTER_TEST(...)`:

```cpp
// previous ctor:
CONCAT(_register_struct_, __LINE__)()
{ /* called once before main */
    register_function(Name, CONCAT(_registered_fun_, __LINE__));
}

// new ctor:
CONCAT(_register_struct_, __LINE__)()
{ /* called once before main */
    auto& test = tf::register_test(CONCAT(_registered_fun_, __LINE__));

    using namespace tf::config;
    tf::do_configure(test, __VA_ARGS__);
}
```

Here we can see why `do_configure` also sets the name: if the variadic part would not include the name (e.g. `#define REGISTER_TEST(Name, ...)`), then the `do_configure` would not work when no options are passed: `tf::do_configure(test, );` is not valid C++.
There are compiler extensions that make `tf::do_configure(test,##__VA_ARGS__);` behave as desired and in C++20 one can use `__VA_OPT__(,)`.

With `using namespace tf::config` we can directly pass `disabled` instead of `tf::config::disabled`, while `tf::do_configure(test, __VA_ARGS__)` applies all options to the test (and sets the name).

This mechanism is extensible because the `tf::config` namespace is a customization point.
New options can be added, even by end-users of this library.
They are found by straightforward overload resolution.
In its current version, new options can also be added in custom namespaces as long as an appropriate `configure` can be found via [ADL](https://en.cppreference.com/w/cpp/language/adl).


## Macro-Free Version

Finally, inspired by the "lambda trick" of a [previous section](#adding-parameters), we can also create a macro-free version that has very little boilerplate.
Using a templated constructor, we only need to define a single variable for registration.

We have a common test framework header:

```cpp
namespace tf
{
struct test
{
    template <class F>
    test(std::string_view name, F&& f)
    {
        // registration code as before ...
    }
};
}
```

and then for each test in some source file:

```cpp
static auto my_test = tf::test("my test", [] {
    // test code
});
```

Similar to a [std::lock_guard](https://en.cppreference.com/w/cpp/thread/lock_guard), we need to assign a variable name, even if it is never actually used.
Apart from that, this method has surprisingly little "noise" (for a non-macro approach).

In C++20 we can even add back line and file information using [std::source_location](https://en.cppreference.com/w/cpp/utility/source_location).


## Final Version

This version of our "function framework" (namespace `ff`) summarizes everything in this post except the macro-free version.
For demonstration purposes, I assume that the functions to register have a signature of `(int, float)` and can be configured with `some_flag` or `some_cnt(n)`.
We start with the shared framework header that must be included whenever a function should be registered:

```cpp
namespace ff
{
/// function signature that we want to register
using fun_t = void (*)(int, float);

/// wrapper class for a "configured function"
class RegisteredFunction
{
public:
    RegisteredFunction(fun_t f, int line, std::string_view file);

    void setName(std::string_view name);

private:
    std::string_view _name;
    fun_t _fun;
    int _line;
    std::string_view _file;
    // ... more options
};

/// registers a new function
/// returns a reference to the wrapper class so that we can configure it later
RegisteredFunction& register_function(fun_t f, int line, std::string_view file);

/// "config" namespace that contains our named argument approximation
namespace config
{
static constexpr struct some_flag_t
{
} some_flag;

struct some_cnt
{
    explicit some_cnt(int v) : value(v) {}
    int value;
};
}

void configure(RegisteredFunction& f, config::some_flag_t);
void configure(RegisteredFunction& f, config::some_cnt const& cnt);

/// the variadic do_configure sets test name and dispatches options to their configure(f, option)
template <class... Args>
void do_configure(RegisteredFunction& f, std::string_view name, Args&&... args)
{
    f.setName(name);
    (configure(f, std::forward<Args>(args)), ...);
}
}

/// two-step macro concatenation to make CONCAT(a, __LINE__) work
#define CONCAT_IMPL(a, b) a##b
#define CONCAT(a, b) CONCAT_IMPL(a, b)

/// the main registration macro that registers the function on program startup
#define REGISTER_FUN(...)                                                                            \
    namespace detail                                                                                 \
    {                                                                                                \
    /* function we later define */                                                                   \
    static void CONCAT(_registered_fun_, __LINE__)(int, float);                                      \
                                                                                                     \
    namespace /* ensure internal linkage for struct */                                               \
    {                                                                                                \
    /* helper struct for static registration in ctor */                                              \
    struct CONCAT(_register_struct_, __LINE__)                                                       \
    {                                                                                                \
        CONCAT(_register_struct_, __LINE__)()                                                        \
        { /* called once before main */                                                              \
            auto& f = ff::register_function(CONCAT(_registered_fun_, __LINE__), __LINE__, __FILE__); \
                                                                                                     \
            using namespace ff::config;                                                              \
            ff::do_configure(f, __VA_ARGS__);                                                        \
        }                                                                                            \
    } CONCAT(_register_struct_instance_, __LINE__);                                                  \
    }                                                                                                \
    }                                                                                                \
    /* now actually defined to allow REGISTER("name") { ... } syntax */                              \
    void detail::CONCAT(_registered_fun_, __LINE__)
```

Then we have a single translation unit that manages the registered functions:

```cpp
namespace ff
{
/// static vector of all registered functions
/// (while avoiding static initialization order fiasco)
std::vector<std::unique_ptr<RegisteredFunction>>& registered_functions()
{
    static std::vector<std::unique_ptr<RegisteredFunction>> v;
    return v;
}

RegisteredFunction& register_function(fun_t f, int line, std::string_view file)
{
    auto& funs = registered_functions();
    funs.push_back(std::make_unique<RegisteredFunction>(f, line, file));
    return *funs.back();
}

void configure(RegisteredFunction& f, config::some_flag_t)
{
    // set up f ...
}

void configure(RegisteredFunction& f, config::some_cnt const& cnt)
{
    // set up f ...
}
}
```

And finally, new functions can be registered in any file:

```cpp
REGISTER_FUN("my fun")(int a, float b)
{
    // user code
}

REGISTER_FUN("my fun 2", some_flag, some_cnt(17))(int x, float y)
{
    // user code
}
```


## Summary

This post got a bit longer than intended by I hope it still can provide to a broad audience.
The initial motivation was to create our own "test/function registration macros" like popular test frameworks do:

```cpp
TEST("my test case")
{
    auto a = 1 + 1;
    auto b = 2;
    CHECK(a == b);
}
```

The main ingredient was constructors of namespace-level objects.
We then started a journey of avoiding the [static initialization order fiasco](https://en.cppreference.com/w/cpp/language/siof), protecting against accidental [ODR violations](https://en.cppreference.com/w/cpp/language/definition), and concatenating identifiers that are themselves macros.

I would say our original goal was accomplished after we finished the [hygienic macro version](#increasing-macro-hygiene).
If that version satisfies your needs, go for it.

The next sections are basically stretch goals: how to handle parametric functions/tests and how to approximate named arguments.
We even have a macro-free version that shows that, while more convenient, macros are not strictly required to have low-boilerplate function or test registration.

Finally, not all "function registration scenarios" should be handled using one of the presented techniques.
This post describes _decentralized_ systems for registration, which makes it easy and convenient to add new entries, i.e. something desirable for test frameworks.
However, sometimes a central place where all functions are registered is more appropriate.

As always: know your problem and choose the best tool.

Additional discussion and comments on [reddit](https://www.reddit.com/r/cpp/comments/jdgxcl/static_registration_macros_eg_test_casefoo/).

(_Title image from [pixabay](https://pixabay.com/photos/old-cash-register-finance-money-1645813/)_)
