---
layout: post
current: post
cover:  assets/images/noodles.jpg
navigation: True
title: "std::sort multiple ranges"
date: 2020-11-28 02:00:00
tags: [c++]
class: post-template
subclass: 'post'
author: philip
excerpt: "Sorting a range of keys while keeping a range of values in sync."
---

`std::sort` is a great utility.
You can easily sort subranges and provide custom comparison functions.
However, it struggles with the following scenario:

```cpp
std::vector<int> keys = ...;
std::vector<std::string> values = ...;

std::sort(...); // ???
```

We want to sort by `keys` but keep the 1-on-1 correspondence with `values`, i.e. keep the ranges "in sync" during sorting.
A common solution is to allocate a vector of indices, sort these indices, and then apply the resulting permutation.
However, the need for an additional allocation and bad cache locality due to indirection make this a suboptimal solution.

In this post, we will design a custom iterator that allows us to sort the two ranges directly without allocation overhead.
The final usage will be:

```cpp
std::sort(sort_it{          0, keys.data(), values.data()},
          sort_it{keys.size(), keys.data(), values.data()});
```

For clarity of exposition, we will assume `int` keys, `string` values, and contiguous ranges for this post.
Making the technique generic, variadic, and support all random access iterators is left as an exercise for the reader (or an additional blog post).

## The Index Solution

Before we start, let's take a quick look at the mentioned index-based solution:

```cpp
// make indices = {0, 1, 2, ...}
auto indices = std::vector<int>(keys.size());
std::iota(indices.begin(), indices.end(), 0);

// sort indices while comparing keys
std::sort(indices.begin(), indices.end(), [&keys](int a, int b) {
    return keys[a] < keys[b];
});

// apply permutation
auto old_keys = keys; // copy
auto old_values = values; // copy
for (size_t i = 0; i < keys.size(); ++i) 
{
    keys[i] = old_keys[indices[i]];
    values[i] = old_values[indices[i]];
}
```

The first allocation is needed for the temporary `indices` vector.
Slightly more non-obvious is the need for copies of `keys` and `values` before applying the permutation.
A simple `keys[i] = keys[indices[i]];` would yield the wrong result (imagine what happens if `keys[indices[i]]` was already overwritten in a previous loop iteration).

There are ways to avoid the key/value copies but there are quite a bit more involved.
Permutations can be decomposed into disjoint cycles and then applied via a cycling swap for each cycle.
Alternatively, permutations can also be decomposed into a series of transpositions, i.e. a sequence of `std::swap(keys[i], keys[j])`.

## A Custom Iterator

Ok, let's design a non-allocating "multi-sort" solution.

`std::sort` accepts any random access iterator (technically, the iterator must satisfy the [LegacyRandomAccessIterator](https://en.cppreference.com/w/cpp/named_req/RandomAccessIterator) and [ValueSwappable](https://en.cppreference.com/w/cpp/named_req/ValueSwappable) concepts).
Thus, we should be able to write an iterator that basically "bundles" iterators into `keys` and `values`, compares `keys`, and swaps both, thus keeping the ranges in sync. 

```cpp
struct sort_it
{
    size_t index;
    int* keys;
    std::string* values;
};
```

The idea is that the iterator only updates `index` and keeps the pointers to `keys` and `values` unchanged.
Iterators need to provide a certain set of operations, [LegacyRandomAccessIterator](https://en.cppreference.com/w/cpp/named_req/RandomAccessIterator) a whole zoo of them.

First, we define some types in our `sort_it`:

```cpp
using iterator_category = std::random_access_iterator_tag;
using difference_type = int64_t;
using value_type = ???;
using pointer = value_type*;
using reference = ???;
```

## The Reference and Value Problem

Usually, `value_type` is some type `T` and we would choose `reference` to be `T&`.
However, in our case, this doesn't work.
`value_type` would be something like `pair<int, string>`.
But `reference` cannot be `pair<int, string>&`, as we have two separate arrays, not a single array of pairs.
Thus, we have to roll our own, separate types for them.
Let's call these `val` and `ref`:

```cpp
struct val
{
    int key;
    std::string value;
};

struct ref
{
    int* key;
    std::string* value;
};

struct sort_it 
{
    ...
    using value_type = val;
    using reference = ref;
    ...
};
```

`std::sort` uses more than one sorting algorithm in most implementations.
Usually, we see some `O(n log n)` algorithm (e.g. quicksort) for large ranges and an `O(n^2)` algorithm (like insertion sort) for small ranges, as they tend to be faster for small `n` in practice.
Some of these use `swap`, some do not.
When reading the gcc implementation, I found that we need to support the following uses:

```cpp
// case A:
std::iter_swap(it_a, it_b);
// ... which calls:
swap(*it_a, *it_b);

// case B:
value_type v = std::move(*it);
*it = std::move(*other_it);
*other_it = std::move(v);
```

Our iterator returns `ref` via `sort_it::operator*()`.
This is used directly in `swap`, so we need to provide a `swap(ref, ref)`.
Note that `ref` is passed by value and cannot be `ref&` as `*it` is not an lvalue.
The desired semantic is, of course, to swap where `ref` points to, not the pointers in `ref` themselves.
We provide `swap` via [(hidden) friend](https://en.cppreference.com/w/cpp/language/friend) in `ref`:

```cpp
struct ref
{
    ...

    friend void swap(ref a, ref b)
    {
        using std::swap;
        swap(*a.key, *b.key);
        swap(*a.value, *b.value);
    }
};
```

That solves case A.

Case B needs a bit more attention.

First, we have `value_type v = std::move(*it);`, where `val` has to be implicitly constructible from `ref&&`.
The intention is to move a value temporarily out of the range and later "return" it via `*other_it = std::move(v);`. 
This implicit conversion can be simply provided by a [user-defined conversion operator](https://en.cppreference.com/w/cpp/language/cast_operator):

```cpp
struct ref
{
    ...

    operator val() && { return {std::move(*key), std::move(*value)}; }
};
```

Note the `&&` at the end of the signature.
This is a [ref-qualified member function](https://en.cppreference.com/w/cpp/language/member_functions#ref-qualified_member_functions) and basically means that the `ref` to `val` conversion is only allowed for rvalue `ref&&`s.
Thus, we can safely move `key` and `value` into our `val` and no `std::string` was copied in the process.

Alternatively, we could have added a `val(ref&&)` constructor to `val`.

The "inverse" operator of `ref&&` to `val` is done at `*other_it = std::move(v);`.
This is a [simple assignment operator](https://en.cppreference.com/w/cpp/language/operator_assignment), accepting `val&&`:

```cpp
struct ref
{
    ...

    ref& operator=(val&& v)
    {
        *key = std::move(v.key);
        *value = std::move(v.value);
        return *this;
    }
};
```

There is one, slightly weird assignment left: `*it = std::move(*other_it);`.
Here, we assign `ref&&` to `ref` and the semantics is to move what `*other_it` points to into what `*it` points to.
Implementation-wise, this is another simple assignment:

```cpp
struct ref
{
    ...

    ref& operator=(ref&& r)
    {
        *key = std::move(*r.key);
        *value = std::move(*r.value);
        return *this;
    }
};
```

And with this, we have finally modelled the "reference semantics" of `ref` and proper interactions with `val`.

## Remaining Operators

There are two classes of operators missing.

First, we want properly defined default comparison, i.e. by default, `std::sort` should sort by `operator<` of our keys.
Unfortunately, almost all combinations are `ref` and `val` are actually used.
We have `*it_a < *it_b`, `v < *it`, and `*it < v`.
The only missing combination is `val < val`.
Our implicit conversion from `ref&&` to `val` is moving values, which we obviously don't want for a comparison.
Thus, we simply define all needed combinations:

```cpp
bool operator<(ref const& a, val const& b)
{
    return *a.key < b.key;
}

bool operator<(val const& a, ref const& b)
{
    return a.key < *b.key;
}

bool operator<(ref const& a, ref const& b)
{
    return *a.key < *b.key;
}
```

Secondly, we need the previously mentioned zoo of random access iterator operators.
The only slightly interesting one in our case is `operator*` for producing `ref`:

```cpp
struct sort_it
{
    ...

    ref operator*() { return {keys + index, values + index}; }
};
```

All the others are the typical `==`, `!=`, `+`, `-`, `++`, `--`, `<`, etc. that you'd expect of a random access iterator.
I was too lazy to implement all of them and only did what I needed for `std::sort` on gcc:

```cpp
struct sort_it
{
    ...

    bool operator==(sort_it const& r) const { return index == r.index; }
    bool operator!=(sort_it const& r) const { return index != r.index; }

    sort_it operator+(difference_type i) const { return {index + i, keys, values}; }
    sort_it operator-(difference_type i) const { return {index - i, keys, values}; }

    difference_type operator-(sort_it const& r) const 
    { 
        return difference_type(index) - difference_type(r.index); 
    }

    sort_it& operator++()
    {
        ++index;
        return *this;
    }
    sort_it& operator--()
    {
        --index;
        return *this;
    }

    bool operator<(sort_it const& r) const { return index < r.index; }
};
```

In C++20, we would need to implement all operators as the [associated concept](https://en.cppreference.com/w/cpp/named_req/RandomAccessIterator) would be checked.
Of course, in a production-grade implementation, we would also implement them all.

And with this, we're done.

Now, the example in the introduction works:

```cpp
std::vector<int> keys = ...;
std::vector<std::string> values = ...;

std::sort(sort_it{          0, keys.data(), values.data()},
          sort_it{keys.size(), keys.data(), values.data()});
```

## Final Version

A fully working example can be found [here on godbolt](https://godbolt.org/z/fb76e1).
I've added a `main` function, so be sure to check the output.

Our custom iterator, value, and reference type is below 100 LOC, so it's almost a compact solution!

```cpp
struct val
{
    int key;
    std::string value;
};

struct ref
{
    int* key;
    std::string* value;

    ref& operator=(ref&& r)
    {
        *key = std::move(*r.key);
        *value = std::move(*r.value);
        return *this;
    }

    ref& operator=(val&& r)
    {
        *key = std::move(r.key);
        *value = std::move(r.value);
        return *this;
    }

    friend void swap(ref a, ref b)
    {
        std::swap(*a.key, *b.key);
        std::swap(*a.value, *b.value);
    }

    operator val() && { return {std::move(*key), std::move(*value)}; }
};

bool operator<(ref const& a, val const& b)
{
    return *a.key < b.key;
}

bool operator<(val const& a, ref const& b)
{
    return a.key < *b.key;
}

bool operator<(ref const& a, ref const& b)
{
    return *a.key < *b.key;
}

struct sort_it
{
    using iterator_category = std::random_access_iterator_tag;
    using difference_type = int64_t;
    using value_type = val;
    using pointer = value_type*;
    using reference = ref;

    size_t index;
    int* keys;
    std::string* values;

    bool operator==(sort_it const& r) const { return index == r.index; }
    bool operator!=(sort_it const& r) const { return index != r.index; }

    sort_it operator+(difference_type i) const { return {index + i, keys, values}; }
    sort_it operator-(difference_type i) const { return {index - i, keys, values}; }
    
    difference_type operator-(sort_it const& r) const 
    { 
        return difference_type(index) - difference_type(r.index); 
    }
    
    sort_it& operator++()
    {
        ++index;
        return *this;
    }
    sort_it& operator--()
    {
        --index;
        return *this;
    }

    bool operator<(sort_it const& r) const { return index < r.index; }

    ref operator*() { return {keys + index, values + index}; }
};
```

## Summary

We set out to write a custom iterator that is able to sort two ranges in parallel, treating one as the "key" and the other as "value" that has to be kept in sync.
`swap` alone was not sufficient to get the correct behavior, as some parts of the `std::sort` implementation operate on `value_type` and `reference` directly.
Thus, we created custom `ref` and `val` structs and implemented all required operators to model the proper semantics.
Note that we could not use `val&` as the reference type because our ranges are independent and do not store elements of type `val`.

The result is a custom random access iterator `sort_it` that keeps the two ranges in sync, as can be seen in [this example on godbolt](https://godbolt.org/z/fb76e1).

For a production-grade implementation, there are a few additional concerns:

* all required operators for the [random access iterator](https://en.cppreference.com/w/cpp/named_req/RandomAccessIterator) should be implemented
* `sort_it`, `ref`, and `val` should be templated on key and value type
* `sort_it` and `ref` should not use pointers, but other random access iterators
* the whole system could be made variadic and support arbitrary many value ranges

(_Title image from [pixabay](https://pixabay.com/photos/noodles-pasta-colorful-pasta-food-1312384/)_)
