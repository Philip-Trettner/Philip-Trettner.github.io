---
layout: post
current: post
cover:  assets/images/menger-recursion.jpg
navigation: True
title: Recursive Lambdas in C++
date: 2020-09-12 02:00:00
tags: [c++]
class: post-template
subclass: 'post'
author: philip
excerpt: 'Ever wondered how to make our beloved [](){}s call themselves?'
---

```cpp
auto fib = [](int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
};
auto i = fib(7);
```

If only it were that simple.

Obviously, any performance-conscious programmer will compute Fibonacci numbers iteratively (or even [explicitly](https://en.wikipedia.org/wiki/Fibonacci_number#Closed-form_expression)), but this solution will serve as an example for an underappreciated tool: _recursive lambdas_.

Lambdas are one of my favorite features in any programming language and while I long for a [shorter syntax in C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0573r1.html), I still use them quite ubiquitously, especially for local functions.
They allow us to abstract behavior into a function while still accessing local variables (through captures) and without leaking new names into the surrounding namespace.
While already plenty powerful, sometimes we might want to call a lambda recursively.

The Fibonacci sequence is an artificial example but I encountered plenty scenarios where you just want to traverse some recursive data structure real quick and a recursive lambda would have been the best solution.
But alas, the above example does not compile because the name `fib` is not accessible within the lambda.

> It's funny how the `x` in `int x = x + 1;` refers to the newly declared variable and is basically never what you want but the `fib` in our example does not refer to the declared lambda even though it is exactly what we want.

## A Suboptimal Solution

Before we get to the good stuff, let's examine a common, yet unsatisfactory solution first:

```cpp
#include <functional>

std::function<int(int)> fib;
fib = [&](int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
};
```

Essentially, by declaring `fib` beforehand, we are able to reference it inside the lambda.
However, `fib` now requires an explicit type and as each lambda expression has its own compiler-generated type, you'll have a hard time naming it (it's a kind of [Voldemort Type](http://videocortex.io/2017/Bestiary/#-voldemort-types)).
Instead, an `std::function` is often the go-to type to store lambdas.

So, why do I consider this solution inferior?

* first of all, look at [the assembly](https://godbolt.org/z/fTTj7r)! A monster, compared to [a normal recursive function](https://godbolt.org/z/3E5anW)
* `std::function` is type erased and often allocates (though some standard libraries perform _small function optimization_ and don't allocate if the size of the lambda is small, i.e. it doesn't capture too much)
* `<functional>` is a big and costly header, basically [costing 200ms+](https://artificial-mind.net/projects/compile-health/) just to include it
* it cannot be made `constexpr`
* it requires writing the function signature twice


## Generic Lambdas to the Rescue??

Let me present my preferred solution:

```cpp
auto fib = [](int n, auto&& fib) {
    if (n <= 1) return n;
    return fib(n - 1, fib) + fib(n - 2, fib);
};
auto i = fib(7, fib);
```

Oof. A generic lambda? Templates? Calling `fib` with itself?

Let me explain!

So, the problem with our opening example was that `fib` is not a visible name inside the lambda.
We simply remedied that by passing `fib` as an additional parameter.
Of course, we don't know the type of `fib` yet, so we use `auto&&` and turn it into a generic lambda.
Also, no, `decltype(fib)&&` wouldn't work.
If we could access `fib`, we wouldn't have this problem in the first case!
Finally, because we now have an additional parameter, we have to pass `fib` to itself every time we call it.

This solution has none of the disadvantages of the previous solution.
Compared to [a normal recursive function](https://godbolt.org/z/3E5anW), we have [one additional jump in the assembly](https://godbolt.org/z/fG3TdW) and of course the slight syntactical inconvenience of having to pass an additional parameter.

If you use the recursive lambda many times in the remainder of the function you can simply wrap it again to make the call more natural:

```cpp
auto f = [&](int n) { return fib(n, fib); };
auto i = f(7);
```

Still produces [good assembly](https://godbolt.org/z/86WdEr).


## Desugaring the Lambda

Okay, okay, I get it.
This might still be too much magic to fully comprehend how the lambda works.
Is it instantiated for every recursion depth?
How would this work with arbitrary deep recursions?
Something is not making sense here.

A step back.

Lambdas are not a magical feature.
They are simply syntactical sugar for a local `struct` that has an `operator()` and each capture as a member (capturing per reference creates reference members):

```cpp
auto k = 7;
auto f = [k](int n) { return n + k; };
return f(3);
```

is basically equivalent to:

```cpp
auto k = 7;
struct lambda_obj
{
    int k; // captured by value
    int operator()(int n) const { return n + k; }
};
auto f = lambda_obj{k};
return f(3);
```

Our _recursive lambda_ is a bit more complex, but not much.
Generic lambdas simply have a templated `operator()`, the rest is the same:

```cpp
auto fib = [](int n, auto&& fib) {
    if (n <= 1) return n;
    return fib(n - 1, fib) + fib(n - 2, fib);
};
auto i = fib(7, fib);
```

is basically equivalent to:

```cpp
struct lambda_obj
{
    template <class F>
    int operator()(int n, F&& fib) const
    {
        if (n <= 1) return n;
        return fib(n - 1, fib) + fib(n - 2, fib);
    }
};
auto fib = lambda_obj{}; // no capture
auto i = fib(7, fib);
```

The only reason you cannot do this in practice is that function-local templates (be it function templates or class templates) are forbidden.
Generic lambdas have a special exemption from that rule.

This also solves the question of the infinite instantiation:
The only template that is instantiated is the templated function `lambda_obj::operator()` and its only instantiation is `int lambda_obj::operator()<lambda_obj>(int n, lambda_obj& fib) const`.
Calling `fib` inside this function is actually the same instantiation! (`fib` still has the type `lambda_obj&`)


## Another Example: Tree Recursion

Okay, that's cool and all, but how does it help in the real life?

Let's say we have a simple recursive data structure, for example a [BSP tree](https://en.wikipedia.org/wiki/Binary_space_partitioning) stored embeddedly in an `std::vector` (or some other contiguous container) for memory efficiency:

```cpp
struct node // only represents inner nodes
{
    // dividing plane
    tg::vec3 plane_normal;
    float plane_distance;

    // idx for child on positive / negative side
    int child_pos;
    int child_neg;

    bool is_on_positive_side(tg::pos3 p) const 
    { 
        return dot(p, plane_normal) > plane_distance; 
    }
};
```

The two members `child_pos` and `child_neg` store the topological information of the tree.
If they are positive, they point to another inner node.
If they are negative, they point into leaf data (stored as "negative leaf idx - 1").

### Point Queries

The first example function is a _point query_, i.e. given a 3D position, return the data stored in the leaf cell:

```cpp
template <class LeafT>
LeafT& get_data_at(std::span<node const> nodes, std::span<LeafT> leaf_data, tg::pos3 p)
{
    auto recurse = [&](int node_idx, auto&& recurse) -> LeafT& {
        if (node_idx < 0) // leaf node
            return leaf_data[1 - node_idx];

        // visit proper child
        auto const& n = nodes[node_idx];
        recurse(n.is_on_positive_side(p) ? n.child_pos : n.child_neg, recurse);
    };
    return recurse(0, recurse);
}

// usage:
std::vector<node> nodes = ...;
std::vector<float> data = ...;
tg::pos3 query_pos = ...;

// NOTE: template arg cannot be deduced 
//      (because the compiler does not know vector<float> corresponds to span<float>)
auto& d = get_data_at<float>(nodes, data, query_pos);
```


### Visitor / Internal Iteration

The second example is a generic traversal operator that takes a direction and a callback.
The callback function is called for all leaf indices ordered ascendingly by the given direction.
This is for example useful to implement the [painter's algorithm](https://en.wikipedia.org/wiki/Painter%27s_algorithm) with render jobs stored in the BSP.

```cpp
// callback signature: (int leaf_idx) -> void
template <class F>
void visit_in_direction(std::span<node const> nodes, tg::vec3 dir, F&& callback)
{
    auto recurse = [&](int node_idx, auto&& recurse) -> void {
        if (node_idx < 0) // leaf node
        {
            callback(1 - node_idx);
            return;
        }

        auto const& n = nodes[node_idx];
        if (dot(n.plane_normal, dir) > 0) // points in same direction
        {
            recurse(n.child_neg, recurse);
            recurse(n.child_pos, recurse);
        }
        else // points in different direction
        {
            recurse(n.child_pos, recurse);
            recurse(n.child_neg, recurse);
        }
    };
    recurse(0, recurse);
}

// usage:
std::vector<node> nodes = ...;
tg::vec3 view_dir = ...;

visit_in_direction(nodes, view_dir, [&](int leaf_idx) {
    // render / process leaf_idx
});
```

Note: the trailing return type `-> void` seems to be mandatory here, otherwise my clang complains that it cannot deduce the return type.


## Conclusion

... or rather a late TL;DR?

Our goal was to make the following _recursive lambda_ work:

```cpp
auto fib = [](int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
};
auto i = fib(7);
```

While this is not directly possible, we can get really close by just adding a parameter!

```cpp
auto fib = [](int n, auto&& fib) {
    if (n <= 1) return n;
    return fib(n - 1, fib) + fib(n - 2, fib);
};
auto i = fib(7, fib);
```

The recipe is simple:
If you want to call a lambda recursively, just add an `auto&&` parameter taking the function again and call that.
This produces basically optimal assembly and can be used in combination with capturing.

Discussion and comments on [reddit](https://www.reddit.com/r/cpp/comments/irupel/recursive_lambdas_in_c/).

### Update 2020-09-13:

If the lambda does not capture anything, it can be declared `static` and the following works:

```cpp
using fib_t = int(*)(int);
static fib_t fib = [](int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
};
auto i = fib(7);
```

Note that `auto` does not work here because the compiler needs to know the type of `fib` before calling it.

(_Title image from [pixabay](https://pixabay.com/illustrations/menger-fractal-design-cube-702863/)_)
