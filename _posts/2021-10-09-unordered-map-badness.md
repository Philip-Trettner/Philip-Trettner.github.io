---
layout: post
current: post
cover:  assets/images/tangled-map.jpg
navigation: True
title: "Measuring std::unordered_map Badness"
date: 2021-10-09 02:00:00
tags: [c++, optimization]
class: post-template
subclass: 'post'
author: philip
excerpt: "When is a good hash a good hash?"
---

I probably could have generated more clicks by titling this "How I Improved `std::unordered_map` Performance By 50x With This Little Trick".

In reality, this post is mostly about my journey crafting the following snippet that allows you to measure the quality of your hash function on a concrete `std::unordered_map`:

```cpp
template <class Map> 
double unordered_map_badness(Map const& map)
{
    auto const lambda = map.size() / double(map.bucket_count());

    auto cost = 0.;
    for (auto const& [k, _] : map)
        cost += map.bucket_size(map.bucket(k));
    cost /= map.size();

    return std::max(0., cost / (1 + lambda) - 1);
}
```

It measures the expected overhead when working with keys that are already in the map.
0 means that your hash distribution is close to optimal.
In my case, the initial hash function had a badness of 550.


Note: this post is not about [perfect hashes](https://en.wikipedia.org/wiki/Perfect_hash_function), which guarantee zero collisions.
Those are also super interesting, but have different use cases.

# Origin Story

Either from grad school or simply through years of osmosis, we all learned that hash maps are awesome associative containers that offer a staggering O(1) insert, delete, and lookup (albeit with some overhead).

The design space of hash maps is quite large and depending on the use case, the trade-off space can change radically.
`std::unordered_map` is (in)famous for having an API that basically forces implementers to use "buckets with linked lists", also known as _separate chaining_.
Many performance-critical applications swear on _open addressing_, often storing keys and values directly in arrays (either together or separate).
These are often called `flat_`maps.
Many requirements and quality attributes influence which particular type is "best":

* Is pointer stability required? (often rules out `flat_` versions, unless only stability of _value_ is required)
* Can entries be deleted individually? (often not required, removing tombstone handling)
* How big are the keys and values? (I've seen all combinations, even huge-key-small-value is occasionally useful)
* What are the relative frequencies of insert, delete, lookup-with-success, and lookup-without-success?
* Is robustness against adversarial attacks required? (e.g. DoS attacks based on enforced collisions)
* Is the hash collision-free? (keys might share the same bucket, but same hash implies same key)

There is already a large corpus of constructive (and not so constructive) discussion on all these particulars.
Many excellent general-purpose and special-purpose hash map implementations are available.
I've added a few links at the end of this post.

However, before choosing a certain hash map implementation, there is a certain elephant in the room that I found myself investigating.
You see, hash maps require "good hashes".
Everyone knows that.
Benchmarks often work on random input data, which easily map to "good hashes".

I have written a few bad hashes over the years and they are really not an issue.
A really bad hash elevates O(1) to O(n).
If simply inserting 100k entries into a hashmap takes half an hour, the problem basically detects itself.

No, "almost bad hashes" and "less-than-optimal hashes" are the real issue, the silent killers.
I recently investigated a piece of code that was too slow for my tastes, but not critically so.
You know, code that works on a few hundred thousand elements and takes a few seconds.
Not suspicious per se, but it happened to be the next chunk I investigated.
A napkin calculation later, the runtime cost was roughly 2000 CPU cycles per element.
It _felt_ too much for some simple floating point arithmetic, but knowing that a cold read from memory can take 200 cycles, I argued to myself that it might be an issue with `std::unordered_map` as we probably all heard that it's "badly designed" and "too slow".

I was half-way into pulling some Google-grade `flat_map` into the project when, on a whim, I slightly modified my hash function.
Instead of the three `float`s whose bit pattern I scrambled together via `boost::hash_combine`, I discretized the `float`s into `int`s before passing them to `hash_combine`.

The result: 50x improved performance.

# Hash vs. Bucket Collisions

I did not need more evidence that something with the hash was wrong.
Just to provide the context, these were my hash functions:

```cpp
void hash_add(size_t& hash, size_t new_hash)
{
    // taken from boost::hash_combine
    hash ^= new_hash + 0x9e3779b9 + (hash << 6) + (hash >> 2);
}

size_t myhash_float(float x, float y, float z)
{
    size_t h = /* some fixed seed */;
    hash_add(h, std::bit_cast<uint32_t>(x));
    hash_add(h, std::bit_cast<uint32_t>(y));
    hash_add(h, std::bit_cast<uint32_t>(z));
    return h;
}

size_t myhash_int(float x, float y, float z)
{
    size_t h = /* some fixed seed */;
    hash_add(h, int32_t(256 * x));
    hash_add(h, int32_t(256 * y));
    hash_add(h, int32_t(256 * z));
    return h;
}
```

So, 50x performance difference between `myhash_int` and `myhash_float`, eh?

First, let me note that `myhash_float` is not a bad hash per se and it is definitely the more versatile one.
`myhash_int` has many collisions if the inputs are too small or if different keys differ by only a small amount.
In my case, it worked due to the nature of the input data.

I don't know the complete history and rationale of `boost::hash_combine` but I guess it was not designed with `float`s in mind and probably comes from the 32 bit era.
On my real-world data set with 90000 entries, I had 3% hash collisions with `myhash_float` and only 0.2% with `myhash_int`.
While a "real" hash like [`xxHash`](https://github.com/Cyan4973/xxHash) realistically produces no collisions for this number of entries, a few collisions do not explain the large performance gap.

> Side note: `xxHash` and similar hash functions are mainly designed for high throughput when processing larger amounts of data, like complete files or buffers.
> That being said, they still often try to guarantee good performance on small data, like keys for hash maps.
> `xxHash` is also explicitly optimized for good throughput and latency on data consisting of only a few bytes.
> However, its higher hash quality is not free and the overhead can be an order of magnitude slowdown when compared to ad-hoc special-purpose hashes for hash maps, like `myhash_xyz` above.

Back to my issue.
Where do we lose 50x performance if the number of collisions is way too low to justify the difference?

On a 64 bit desktop, hash maps tend to use 64 bit hashes, like a `size_t`.
However, the number of buckets is significantly less, typically within factor 2 of the number of entries.
Thus, each hash map has a way to map a hash `h` to a bucket index `i`.

The naive mapping would be `i = h % bucket_count`.
Full 64 bit division and modulo is quite expensive, requiring 25-40 cycles on typical desktops.
If `bucket_count` is a power of two, we can optimize the mapping to `i = h & (bucket_count - 1)`, which is effectively free.

The discerning reader might already see the problem:
This mapping now amounts to throwing away most of the bits of `h`.

Imagine your key consists of `uint32_t a, b` and your hash is `(a << 32) | b`.
This hash is completely free of collisions.
However, if you have less than 4 billion buckets, then the bucket index will completely ignore `a`, leading to tons of actual collisions for keys that differ only in `a`.

As said earlier, these obvious cases are usually trivial to detect.
In my case, I witnessed a partial quality degradation of the `key -> hash -> idx` mapping.
The input `float`s came from decompressed 3D positions, so they only had a small range of exponents and a few mantissa patterns that really appeared.
With `boost::hash_combine`, this was _somehow_ assembled into a 64 bit hash.
The 3% _hash_ collision rate probably means that `hash_combine` did a less-than-perfect job scrambling the `float` patterns.
However, the real killer came from `std::unordered_map`, mapping the hash to a bucket index.
It turned out that more than 98.6% of the keys had "bucket collisions", i.e. had to share their bucket with other keys.
With `myhash_int`, this was still 86.2%.


# Optimal Behavior

Before trying to quantify how bad my first hash was, let's briefly talk about what is the best-case scenario.
Zero bucket collisions are the realm of perfect hashes, which require heavy precomputation and in general only work with known input data.

If only the input distribution (not the actual data) is known, we want a `key -> idx` mapping that is _uniform_.
Without getting too fancy in the math: if someone hands you a `key` drawn from the input distribution, you want to hand back an `idx` that has a roughly uniform distribution over `0 .. bucket_count-1`.

In reality, the user is responsible for the `key -> hash` mapping, while the hash map provides the `hash -> idx` mapping.
Some cooperation is required to make the total mapping high-quality.
In theory, the hash map could always provide a strong `hash -> idx` mapping, e.g. via `xxHash`, but that is usually considered net-negative for performance.
If the input data is already sufficiently uniform, always paying for this extra hashing is extremely wasteful.

So, assuming a uniform mapping, what is the expected number of bucket collisions?

Well, that depends on the [load_factor](https://en.cppreference.com/w/cpp/container/unordered_map/load_factor), i.e. the ratio of input keys to buckets.
If the `load_factor` is 1, then we have an equal number of keys and buckets.
Here, on average, 37% of buckets are empty, another 37% of buckets have exactly 1 key, 18% have 2 keys, 6% have 3, and 2% have 4 or more keys.
This follows a [Poisson distribution](https://en.wikipedia.org/wiki/Poisson_distribution) where the load factor is lambda.

What load factor is optimal is a different discussion, but given a fixed load factor (typically between 0.5 and 1.0), if our hash function produces roughly the same number of collisions as the corresponding Poisson distribution predicts, I would consider it "optimal enough".


# Measuring Badness

So, how does my hash compare to an optimal one?
How do we measure that?

It turns out that `std::unordered_map` has a [bucket API](https://en.cppreference.com/w/cpp/container/unordered_map#Bucket_interface):

```cpp
std::unordered_map<Key, Value, Hasher> my_map;

// number of buckets
size_t bcnt = my_map.bucket_count();

// bucket index from key
size_t bi = my_map.bucket(some_key);

// number of keys in this bucket
size_t bsize = my_map.bucket_size(bi);
```

With this API, we can measure how close (or far away) we are from the Poisson distribution.
For each key in the map, we sum up the corresponding `bucket_size`.
Divided by the number of keys, this is the average bucket size from the perspective of a key.
Half of this is the expected number of comparisons needed when looking up the key.

In the optimal case, `load_factor` is the average number of keys in a bucket (the expected value of the Poisson distribution).
However, when we know that a given key is part of the map, this average increases to `1 + load_factor`.

The result is this little snippet:

```cpp
template <class Map> 
double unordered_map_badness(Map const& map)
{
    auto const lambda = map.size() / double(map.bucket_count());

    auto cost = 0.;
    for (auto const& [k, _] : map)
        cost += map.bucket_size(map.bucket(k));
    cost /= map.size();

    return std::max(0., cost / (1 + lambda) - 1);
}
```

I was too lazy to template it on all 5 types required for `std::unordered_map`.
Officially, I call that a feature because now it supports all types with a compatible bucket API.

The return value is slightly remapped.
`cost / (1 + lambda)` is the relative cost factor to the optimal distribution.
We subtract 1 and clamp it to 0 from below so it's a bit easier to read:
A badness of roughly 0 means that the current bucket distribution is close to optimal.
1 means that on average 100% more comparisons than optimal are required.

# "Fixing" my Issue

Turns out, `myhash_float` has a badness of 550 on my data. Outch.

Of course, this does not mean that the performance gap to the optimal case is a factor of 550.
Similar to [Amdahl's law](https://en.wikipedia.org/wiki/Amdahl%27s_law), this factor is only realized if the program literally does nothing else (and we ignore caching and other effects that reality pesters us with).

`myhash_int` still has a badness of 1.3.

Not optimal, but so much better that my program sped up by a factor of 50+.

`xxHash` directly on my 3 input `float`s has a badness of 0.

Interestingly enough, using the result of `myhash_float` as a seed for a single round of [xorshift](https://en.wikipedia.org/wiki/Xorshift) has a badness of 0.04.
So, even if `myhash_float` has 3% innate collision rate, a cheap scrambling at the end is all it takes to get near optimal hash distribution.

# Conclusion

In my opinion, hash maps are among the most interesting data structures.
The only container that is more useful is the array, but apart from abstractions like `vector` and `span`, arrays are quite compact in design space.
Hash maps have myriads of useful variants and tradeoffs.

This post is about the practical quality of hash functions and how to measure them.
With the bucket API of `std::unordered_map`, we can actually quantify how far from optimal our concrete map is.

My little snipped can be adapted to hash maps with open addressing by measuring the number of comparisons needed until a key is found.
This is usually not an exposed metric, though I suppose one could simply write a counting equality comparer for that.

# Further Reading

* [Benchmark of major hash maps implementations (2016)](https://tessil.github.io/2016/08/29/benchmark-hopscotch-map.html)
* [I Wrote The Fastest Hashtable (2017)](https://probablydance.com/2017/02/26/i-wrote-the-fastest-hashtable/)
* [Hashmaps Benchmarks - Overview (2019)](https://martin.ankerl.com/2019/04/01/hashmap-benchmarks-01-overview/)

(_Title image from [pixabay](https://pixabay.com/photos/europe-travel-map-world-1264062/)_)
