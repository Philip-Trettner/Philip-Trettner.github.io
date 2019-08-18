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
std::vector<triangle> vertex_clustering(std::span<triangle const> mesh)
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
    std::vector<triangle> result;
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
* `std::map` and `std::set` are probably slower than their `unordered_` versions if the mesh is large, though we will roll our own hash structure later on anyways.
* We know the size of the `result` vector and could easily call `.reserve(output_triangles.size())`. Even faster would be to pass the `result` vector by reference (to potentially reuse memory), `.resize` it to the desired size and directly access the elements we want (to bypass the "do I have to increase capacity?" check in `.emplace_back`).

TODO: images


## Minimizing Quadrics

TODO: images of different representative points (center, centroid, quadric)


## Custom Hash Maps

TODO: benchmarks


## Going Further

In practice you want to make sure that the same triangle is not emitted multiple times.
Changing `output_triangles` into an `unordered_set` is a first step but doesn't cover permutations of the same vertices.
The typical approach to ensure uniqueness of sets would be to sort the `std::array<ipos3, 3>` of vertices.
However, that could invert the vertex order and thus flip the triangle.
Instead we can choose the cyclic permutation with the smallest first vertex.

Another stumbling block when using vertex clustering in practice is that vertices typically are not only positions but also have attributes like colors, normals, or texture coordinates.
This is a problem for all decimation schemes and still topic of ongoing research.
A common term to search for is "appearance-preserving simplification".
Sometimes it's sufficient to add quadrics for each attribute and take their optimal values for the representative point.
This usually works if the attributes behave roughly linearly but fails hard in cases where the relationship is more complex, e.g. across cuts in the texture domain.


## Conclusion


## TODO

* images
