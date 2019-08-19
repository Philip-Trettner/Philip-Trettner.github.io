---
layout: post
current: post
cover:  assets/images/construction.jpeg
navigation: True
title: "Fast Vertex Clustering"
date: 2019-08-17 01:00:00
tags: [geometry, c++, optimization]
class: post-template
subclass: 'post'
author: philip
excerpt: TODO
---

3D meshes often have huge amounts of vertices and must be decimated for use in games or other applications.
Two popular algorithms for this are _incremental decimation_ and _vertex clustering_.
In this post I'll explain:
* the idea behind vertex clustering
* how to find the best representative point using quadrics
* how to make it as fast as possible using a custom hash map

TODO: gif

## Vertex Clustering

The vertex clustering algorithm is surprisingly easy.

Imagine laying a 3D voxel grid through your triangle mesh.
All vertices contained in the same grid cell are collapsed to a single point.

Each original triangle falls into one of three categories:
1. All three vertices are in the same cell and the triangle collapses into a point
2. Two of the vertices are in the same cell and the triangle degenerates into a line
3. All three vertices are in different cells

Only triangles of the third category generate output (and only if the specific combination of vertices was not already encountered).

{% raw %}
```cpp
struct pos3 { float x, y, z; };
struct ipos3 { int x, y, z; };
struct triangle { pos3 v0, v1, v2 };

std::vector<triangle3> vertex_clustering(std::span<triangle3 const> mesh)
{
    std::map<ipos3, cell> cells;
    std::vector<std::array<ipos3, 3>> output_triangles;
    
    // process all triangles
    for (auto const& t : mesh)
    {
        // compute "rounded" grid positions for each vertex
        auto const i0 = grid_pos(t.v0);
        auto const i1 = grid_pos(t.v1);
        auto const i2 = grid_pos(t.v2);

        // add vertices to cells
        cells[i0].add(t.v0);
        cells[i1].add(t.v1);
        cells[i2].add(t.v2);

        // if case 3, remember triangle
        if (i0 != i1 && i1 != i2 && i2 != i0)
            output_triangles.emplace_back({{i0, i1, i2}});
    }

    // compute representative points
    for (auto& [_, c] : cells)
        c.compute_representative_point();

    // generate output triangle soup
    std::vector<triangle3> result;
    for (auto const& [i0, i1, i2] : output_triangles)
    {
        auto const p0 = cells.at(i0).representative_point();
        auto const p1 = cells.at(i1).representative_point();
        auto const p2 = cells.at(i2).representative_point();

        result.emplace_back(p0, p1, p2);
    }

    return result;
}
```
{% endraw %}

And that's the whole algorithm.

This code is written for maximal readability and there is some low-hanging fruit for making it faster.
* We are already using [`std::span`](https://en.cppreference.com/w/cpp/container/span) (officially C++20 but many backports available) for the mesh to support any data-contiguous input.
* `std::map` is probably slower than its `unordered_` version if the mesh is large, though we will roll our own hash structure later on anyways.
* We know the size of the `result` vector and could easily call `.reserve(output_triangles.size())`. Even faster would be to pass the `result` vector by reference (to potentially reuse memory), `.resize` it to the desired size and directly access the elements we want (to bypass the "do I have to increase capacity?" check in `.emplace_back`).

TODO: images


## Minimizing Quadrics

TODO: images of different representative points (center, centroid, quadric)


## Custom Hash Maps

At this point our method is already giving results of reasonable quality.

But how fast is it?

TODO: benchmarks

Minimizing the error quadric requires solving a system of linear equations.
However, the system is of size $3 \times 3$ and due to our regularization we can safely solve it by computing $A^{-1} b$ which takes roughly 20 to 30 cycles on a modern PC.

The main bottleneck of this method is accessing the `std::map`.

Often, vertex clustering is introduced by using a literal grid (3D array) of cells.
Accessing a cell would amount to an indexed array lookup.
However, this undermines any runtime guarantee as the mesh could be an arbitrarily bad fit for a grid and create many unused cells.
It is also an easy way to blow up memory consumption.
On the other hand, using an associate container guarantees that no more than $3n$ cells are created given $n$ input triangles.

Hashmaps are an incredibly versatile data structure and probably one of the hardest to "get right" or "optimal" in the general case.
While `std::unordered_map` is (IN DIE JAHRE GEKOMMEN), some popular high-performance general-purpose hashmaps are:
* Google's shit
* ska hashmap
* flatmap
* TODO!

Under the hood, `std::unordered_map<K, V>` is basically a `std::vector<std::forward_list<std::pair<K, V>>>`.
The key is hashed and used as an index (think `i % vector.size()`) into the vector.
Each "bucket" of the vector contains a singly-linked list of key-value-pairs.
TODO: verify

As with any data-intensive problem, the main performance problems are allocations and data locality / access patterns.
Here is a rough list of ideas what we can learn from modern, optimized hashmaps and how we could beat them:
* It is usually a better idea to store keys and values separately because lookups usually check multiple keys but return only one value
* `std::forward_list` separately allocates each node (Want to trigger game developers? Use a `std::list` in performance-sensitive code.)
* Our algorithm never removes entries and actually has two phases: a build phase and a lookup-only phase
* General-purpose hashmaps don't know the distribution of keys and hashes and must be perform well on a greater range of scenarios. We can tailor the hash function to our data.
* Hashmaps can also come in the _open addressing_ variety where at most one key is stored per bucket and hash collisions are resolved through _probing_, i.e. testing a deterministic sequences of buckets after the first one to look for the next empty slot.
* Usually, hashmaps have a certain load factor (ratio of empty to used buckets) in which they perform optimally. When the load factor crosses a configurable threshold while adding elements, the hashmap is rebuilt with a larger number of buckets.

Armed with these ideas, let's see if we can beat the general-purpose hashmaps!

The plan:
* an open addressing hashmap
* storing keys and values in separate `vector`s
* no support for deleting elements (and thus no need for tombstone entries)
* a tweaked hash function for our scenario
* a cache-local mixture of linear probing and double hashing (TODO: name)

```cpp
struct key
{
    int x;
    int y;
    int z;
    uint32_t hash;
};

struct value // error quadric
{
    float A[6];
    float b[3];
};

static_assert(sizeof(key) == 16);
static_assert(sizeof(value) == 36);
```

The hashmap itself is very simple:

```cpp
struct hashmap
{
    value& operator[]()

private:
    std::vector<key> keys;
    std::vector<value> values;
};
```

TODO: implement me

## Going Further

In practice you want to make sure that the same triangle is not emitted multiple times.
Changing `output_triangles` into an `unordered_set` is a first step but doesn't cover permutations of the same vertices.
The typical approach to ensure uniqueness of sets would be to sort the `std::array<ipos3, 3>` of vertices.
However, that could invert the vertex order and thus flip the triangle.
Instead we can choose the cyclic permutation with the smallest first vertex as the canonical key.

Another stumbling block when using vertex clustering in practice is that vertices typically are not only positions but also have attributes like colors, normals, or texture coordinates.
This is a problem for all decimation schemes and still topic of ongoing research.
A common term to search for is "appearance-preserving simplification".
Sometimes it's sufficient to add quadrics for each attribute and take their optimal values for the representative point.
This usually works if the attributes behave roughly linearly but easily fails in cases where the relationship is more complex, e.g. across cuts in the texture domain.

## Conclusion


## TODO

* images
* formula
* words
