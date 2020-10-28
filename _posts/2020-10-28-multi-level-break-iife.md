---
layout: post
current: post
cover:  assets/images/brake.jpg
navigation: True
title: "Multi-Level Break in C++ via IIFE"
date: 2020-10-28 02:00:00
tags: [c++, fun]
class: post-template
subclass: 'post'
author: philip
excerpt: 'Ever wanted to break more than one level at once?'
---

I guess we all have been at this point.

```cpp
for (auto i : ...)
    for (auto j : ...)
        if (condition(i, j))
        {
            break outer???
        }
```

You want to search something, and for one reason or another you end up with a nested loop.
You find what you searched for and now want to `break` all the way to the outer loop.

If only we had multi-level `breaks`.

But we don't.

So people introduce flags:

```cpp
auto found = false;
for (auto i : ...)
{
    for (auto j : ...)
        if (condition(i, j))
        {
            found = true;
            break;
        }

    if (found)
        break;
}
```

Which introduces quite the clutter and can be error-prone if the loops contain more code and the second `break` is somehow missed or not executed.

Or people (\*gasp\*) introduce `goto`s:

```cpp
for (auto i : ...)
    for (auto j : ...)
        if (condition(i, j))
        {
            goto next;
        }
next:;
```

But C is the enemy, isn't it?

Okay, so we clearly need a multi-level break appropriate for modern C++.

I mean, we could propose new syntax... `co_break`, anyone? 

> Yeah, yeah, I get it. Has been done too many times already.

But behold!
We already have shiny [lambda expressions](https://en.cppreference.com/w/cpp/language/lambda) and [immediately invoked function expressions (IIFEs)](https://en.wikipedia.org/wiki/Immediately_invoked_function_expression).
And they are more than adequate to solve our problem:

```cpp
[&] {
    for (auto i : ...)
        for (auto j : ...)
            if (condition(i, j))
            {
                return;
            }
}();
```

No need to introduce additional flags, identifiers, labels. Just good old `[&]{}();` and `return`.

> This post is tagged as `C++` and `fun` and is suited best as some mid-week entertainment in turbulent times.

(_Title image from [pixabay](https://pixabay.com/photos/auto-repair-workshop-brake-disc-1954643/)_)
