---
layout: post
current: post
cover:  assets/images/free-beer-span-2.webp
navigation: True
title: "A Practical Public Domain std::span"
date: 2024-05-01 02:00:00
tags: [c++]
class: post-template
subclass: 'post'
author: philip
excerpt: "grab your free std::span here!"
---

For me, the most underrated data type in C++ is the `std::span<T>`.
A non-owning view on a contiguous region of memory.
`std::string_view` is to `std::string` what `std::span<T>` is to `std::vector<T>`.
It's the reason `std::vector<T> const&` became an obscurity in my code.

Still, it's a C++20 addition and I have to work on many C++17 projects.
Bringing in new dependencies and coordinating licenses is a hassle.
So here it is, for everyone to use: a hassle-free, public domain, forward-compatible subset of `std::span<T>`:

```cpp
#include <cstddef>
#include <typetraits>

// license: public domain, no warranty implied
namespace cc // adapt as desired
{
template <class T>
struct span
{
    // ctors
public:
    constexpr span() = default;

    constexpr span(T* data, size_t size) : _data(data), _size(size) { }

    constexpr span(T* data, T* data_end) : _data(data), _size(data_end - data) { }

    template <size_T N>
    constexpr span(T (&data)[N]) : _data(data), _size(N) { }
    
    template <class Container, class = std::void_t<
        decltype(static_cast<T*>(std::declval<Container>().data())), 
        decltype(size_t(std::declval<Container>().size()))>>
    constexpr span(Container&& c) : _data(c.data()), _size(c.size()) { }

    constexpr operator span<T const>() const noexcept { return {_data, _size}; }

    // container api
public:
    constexpr T* begin() const { return _data; }
    constexpr T* end() const { return _data + _size; }

    constexpr T* data() const { return _data; }
    constexpr size_t size() const { return _size; }
    constexpr size_t size_bytes() const { return _size * sizeof(T); }
    constexpr bool empty() const { return _size == 0; }

    constexpr T& operator[](size_t i) const { return _data[i]; }

    constexpr T& front() const { return _data[0]; }
    constexpr T& back() const { return _data[_size - 1]; }

    // subviews
public:
    constexpr span first(size_t n) const { return {_data, n}; }
    constexpr span last(size_t n) const { return {_data + (_size - n), n}; }
    constexpr span subspan(size_t off, size_t cnt) const { return {_data + off, cnt}; }
    constexpr span subspan(size_t off) const { return {_data + off, _size - off}; }

    // member
private:
    T* _data = nullptr;
    size_t _size = 0;
};
}
```

It's intended as copypasta:
- copy it into your codebase
- (optional) change the namespace
- (optional) adapt to your coding style
- (optional) add bound check assertions


## Implementation Notes

Looking at [cppreference for `std::span<T>`](https://en.cppreference.com/w/cpp/container/span) quickly shows that this is only a subset of the interface.
This is a deliberate choice to keep the code compact.
Should you need a more fully featured version, try to upgrade to C++20, extend the snippet, or choose any of the dozen of available span/array-view/array-ref helpers out there.

My version is a 80-20 version of what I've needed so far and offers the following trade-offs:
* forward-compatible (once C++20 becomes available in your project, it can be replaced by `std::span<T>` without changing the API)
* no second template argument for fixed size spans
* no typedefs for `xyz_type`, `xyz_iterator`, etc.
* only simple begin/end, no c/r versions
* only a subset of constructors with less strict concept checking
* no deduction guides

The result is a quite readable ~50 LOC practical implementation.

TODO: link to godbolt
TODO: test impl

(_Title image from DALL-E via "a free array view, free as-in beer"_)
