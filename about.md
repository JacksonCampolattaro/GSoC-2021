---
title: "About"
date: 2021-08-18T16:13:27+02:00
tags: []
featured_image: ""
description: ""
---

# Final Report

CGAL’s Axis-Aligned Bounding Box Tree is an acceleration structure 
which speeds up common tasks such as collision-detection. 
It is used both directly and throughout other packages in the library, 
so any performance improvements made to this package will pay dividends elsewhere. 
My original goal for this project was to incorporate SIMD into the package
using techniques borrowed from Intel's own library [Embree](https://www.embree.org/).

Over the course of the project, the central goal shifted.
The causes and effects of this shift are explained in the timeline below,
but the end result is that most of the optimizations I've produced are only partly SIMD related.

One of the primary work products that came out of my endeavor this summer was a modification to the 
tree's approach for construction that can increase performance by as much as 50%.
It uses a technique pioneered by Embree, 
building the tree rapidly by sorting the items it contains along a space-filling-curve.
The relevant pull-request is available to view 
[here](https://github.com/CGAL/cgal/pull/5893),
and documentation for the feature is available 
[here](https://cgal.github.io/5893/v1/AABB_tree/classCGAL_1_1AABB__traits__construct__by__sorting.html).

I'm refining another optimization which uses an implicit tree structure 
for a more compact and SIMD friendly arrangement of the data in memory.
This may become its own pull request in the future.

The AABB-package ultimately makes up a small fraction of CGAL's code base,
so a major part of the value of this project is in the institutional knowledge it produced.
As such, I documented my work thoroughly on CGAL's internal wiki;
I've created a public mirror of that documentation [here](https://jacksoncampolattaro.github.io/GSoC-2021/posts/wiki).

## Timeline

### SIMD Exploration

In the first stage of the project, I wrote quite a bit of exploratory code and got familiar with tooling.
All of the code produced is contained in the 
[_simd-experiments directory](https://github.com/JacksonCampolattaro/cgal/tree/gsoc2021-simd-campolattaro/_simd-experiments)
of the [gsoc2021-simd-campolattaro](https://github.com/JacksonCampolattaro/cgal/tree/gsoc2021-simd-campolattaro)
branch of my fork.
This was wide-ranging work, covering topics spanning from hand-analysis of assembly 
to benchmarking reference implementations of specific algorithms.

Through this work, I identified a variety of approaches for introducing SIMD to CGAL, 
in order from highest to lowest level:
- Compiler autovectorization
- OpenMP Pragma directives
- High level libraries like [Eigen](https://eigen.tuxfamily.org/index.php?title=Main_Page)
- Low level libraries like [XSimd](https://xsimd.readthedocs.io/en/latest/)
- SIMD Intrinsics
- Embedded assembly

During this stage, I explored autovectorization, OpenMP, and low-level libraries the most thoroughly;
I also examined some assembly that these approaches produced.
In the end, I decided that the best way-forward would be to rely on the autovectorizer;
making CGAL's code more SIMD-friendly rather than explicitly adding it myself.

### Combining SIMD with CGAL

The next step was to try integrating some concepts I'd explored into CGAL's code.
This was easier said than done, as CGAL is a very different code-base than something like Embree.
Embree is heavily optimized for a specific use-case (rendering). 
Its code is C-like and low-level, which makes it a good target for SIMD optimizations.
CGAL, on the other hand, uses modern C++ constructs and has heavily templated code.
CGAL's high level of genericity makes vectorization challenging;
optimal vectorized code is different depending on whether the data is in the form of 
floats or doubles, for example, but CGAL supports both.

I had limited success making
[certain functions](https://github.com/JacksonCampolattaro/cgal/blob/c6f7da8764fb37746e81925bc2a9ddd3cd1191d9/Kernel_23/include/CGAL/Bbox_3.h#L191)
more SIMD friendly,
but for others I had trouble enabling vectorization
[without also losing exactness](https://github.com/JacksonCampolattaro/cgal/blob/8cc8fb80c4b01b6b61eca3a593ecd5b2bc4249da/Intersections_3/include/CGAL/Intersections_3/internal/Bbox_3_Segment_3_do_intersect.h#L453).

CGAL's exactness guarantees presented the biggest challenge for vectorization. 
CGAL can use different math kernels to retain high levels of precision,
and to ensure that rounding errors don't propagate to create incorrect results.
Using SIMD in exact computation could be the subject of its own research project.

This challenge caused a shift in the goal of the project, but it wasn't the only reason.
- We determined that SIMD was only one part of Embree's performance advantages,
  if we wanted to match their performance we would need to include some other optimizations.
- The compiler already does a passable job at vectorizing CGAL's code,
  which limits how much improvement we can expect from further intentional vectorization.

### N-Way Tree

One major difference of Embree is the topology of its AABB-tree.
Embree provides an "N-Way Tree", where each node contains N children, rather than only 2.
N is typically set to the size of the CPU's SIMD registers, so that intersection with all children can be done rapidly.

I [modified CGAL's AABB-tree node](https://github.com/JacksonCampolattaro/cgal/blob/n-way-splits/AABB_tree/include/CGAL/internal/AABB_tree/AABB_node.h)
to allow for more than two children in [a new branch](https://github.com/JacksonCampolattaro/cgal/tree/n-way-splits).
The process of converting it was simpler than I expected,
and it came with some positive side effects.
Making construction and traversal functions generic in regard to N actually simplified a lot of the logic.
This is where I discovered an optimization that could be made to the tree's topology:
my changes to the tree incidentally added bounding boxes to the leaf nodes, 
which further simplified traversal, and had a significant positive effect on performance.

Unfortunately, the higher dimensionality of the tree didn't provide a positive affect without the use of SIMD.
This meant that I could produce better performance, but I wasn't able to achieve those results
without using an algorithm that omitted CGAL's exactness guarantees.

### Bounding-box workaround

We decided that if we wanted to take advantage of SIMD we would need to find places exactness wasn't necessary.
My mentor suggested the approach of boxing the queries:
because intersections between bounding boxes don't require full precision, 
they could be used to sidestep the issues that make SIMD difficult.
If we could create a box that surrounds the query, 
we could use that box to cheaply estimate whether the query intersected with another box.
To find this box "conservatively" -- that is, while preserving exactness --
we used CGAL's existing interval-math capabilities.
I worked with my mentor to devise an algorithm that could put a box around only the relevant part of the query,
which let us use smaller boxes as we descended the tree.

After further development, I had produced a prototype which fit nicely into CGAL's existing abstraction.
This work is visible in [another branch](https://github.com/JacksonCampolattaro/cgal/tree/boxed-queries);
it adds a new [Boxed Query type](https://github.com/JacksonCampolattaro/cgal/blob/boxed-queries/AABB_tree/include/CGAL/internal/AABB_tree/Boxed_query.h), 
which wraps the objects we traditionally use for traversing the tree.
This implementation provides several tunable parameters:
- Number of boxes that surround the query (ranging from singly boxed to many)
- Frequency with which we shrink the box to more closely fit the query as we descend the tree
  (ranging from never, to every level, to heuristically determining when it's necessary)
- How reliable we expect the bbox to be at predicting if the query itself intersects
  (either no trust, or assuming that all intersections of boxes represent real intersections)
- Number of children per node, when using an N-way tree
  (The cheaper bbox-bbox intersections might make wider trees worthwhile)
  
This is where benchmarking became a central part of our project.
We didn't do a full grid-search of the parameters, but we did test a wide variety of combinations.
We found some local maxima, for example in some configurations wider trees did indeed become worthwhile,
but ultimately the ideal configurations was clear.
We got the best performance from placing full trust in a single box around the query, 
traversing a 2-way tree and shrinking the box according to our heuristic.

Our tests showed that this approach could consistently produce a 10-20% improvement in performance,
unfortunately refinements in our benchmark showed that 
these improvements were limited to specific types of queries.
Moreover, the best performance was for scenarios that are unusual in CGAL,
and for other scenarios the performance difference was almost negligible.
The core of this issue was that the math required to build a box around a query is similar to the math for 
checking if that query intersects with a box.
Ultimately, if we wanted to get performance from this approach it would be necessary to find a way
to build boxes a lot less often than we performed intersections.

### Construction by Spatial-sorting

We decided to examine another technique used by Embree: construction by Spatial Sorting.
This is an approach I found fascinating: 
where building a tree traditionally means recursively splitting its contents into groups,
we can achieve a similar result by sorting the contents along a space-filling curve like the 
[Hilbert curve](https://en.wikipedia.org/wiki/Hilbert_curve).
This naturally puts the contents in an order such that nearby items are close together in the list,
meaning that they can be easily and efficiently grouped into nodes.

This technique is interesting because it naturally comes with a tradeoff.
The tree can be built faster, but it tends to produce a "lower quality" tree.
This means that the arrangement of the nodes is less optimal,
and traversals will typically be slightly slower.
This means that it's suitable in situations where it's important to build a tree quickly,
or we're not using it for very many queries.
To incorporate this into CGAL's tree, it will have to be optional.

Luckily, CGAL's genericity means the AABB-tree is very modular,
the user can easily replace or change the algorithms or data structures that underly the tree.
Adding spatial sorting was as simple as adding a new construction algorithm for the user to choose from.
The algorithm itself was also relatively simple to implement;
CGAL already provides its own `hilbert_sort()`, 
so the only challenge was to fit it into the same semantics as the other construction algorithm.
This implementation was done on [yet another branch](https://github.com/JacksonCampolattaro/cgal/tree/construct-by-sorting)
as part of [a new class](https://github.com/JacksonCampolattaro/cgal/blob/construct-by-sorting/AABB_tree/include/CGAL/AABB_traits_construct_by_sorting.h).

Once it was working, the performance results were very exciting.
The new algorithm made construction as much as 50% faster,
while only making traversal around 20% slower.
This is better than it sounds, because construction is a much more expensive process than traversal.
I created a [special benchmark](https://github.com/JacksonCampolattaro/cgal/blob/construct-by-sorting/AABB_tree/benchmark/AABB_tree/construction_by_sorting.cpp)
to look at how this performance would affect real (massive) workloads.
We determined that this algorithm would be the best option
unless the user does tens or hundreds of thousands of traversals with the tree,
at which point faster traversals become worthwhile.

### Pull-Request

Now that I had a high-performing optimization
I was ready to [create a Pull Request](https://github.com/CGAL/cgal/pull/5893) 
and begin integrating it into the rest of CGAL.
I already had a benchmark which could demonstrate its advantage,
so I began adding documentation and examples for the new code which will become a part of CGAL's manual.
CGAL's code-review process is thorough,  and it's given me great opportunities to discuss ways to simplify the code
or make the API more ergonomic with other members of the organization.

### Implicit Tree Structure

I'm currently exploring another technique suggested by experts at CGAL, this one unrelated to Embree.
Implicit data structures are data structures where the relationships between 
their contents locations in memory carry implicit meaning.
An implicit tree is one such example:
if you structure your tree properly, 
you can intuit its structure based only on how the nodes are arranged in a flat list.
This can be used to save a lot of space,
because it isn't necessary to store the connections between nodes.

I've implemented a couple of versions of this, experimenting with different approaches.
First I created one 
[built on top of the N-way tree](https://github.com/JacksonCampolattaro/cgal/blob/implicit-tree-structure/AABB_tree/include/CGAL/internal/AABB_tree/AABB_node.h), 
which provided external functions to find the children of a node.
In this version, the nodes contained their bounding box and (optionally)
a reference to the item they represented, if they were a leaf.

A [newer version](https://github.com/JacksonCampolattaro/cgal/blob/implicit-tree-structure-2-way-splits/AABB_tree/include/CGAL/internal/AABB_tree/AABB_node.h) 
takes the concept even further.
I replaced the array of nodes with an array of bounding boxes,
and used a "node handle" type to enable traversing the tree as usual.
This version uses a function to map leaf nodes directly to locations in the primitive list,
so I was able to remove that reference from the nodes, further shrinking them.
It also contains fewer changes to CGAL's existing solution, 
which makes it simpler to merge in the future.

I'm using benchmarks to see how these trees compare to one another,
with a particular interest in 
[how they interact](https://github.com/JacksonCampolattaro/cgal/blob/implicit-tree-structure-2-way-splits-leafless/AABB_tree/include/CGAL/internal/AABB_tree/AABB_node.h)
with the topological change I discovered when I was creating the N-way tree.
These already provide a memory advantage, 
but I'm hoping that because of the improved packing of their data
I can produce a speed advantage too, with the help of SIMD.
In the future I intend to submit another PR with these changes.

> ### Rubric
> 
> The target of the link should contain: 
> - a short description of what work was done, 
> - what code got merged, 
> - what code didn't get merged, 
> - what's left to do
> 
> The best examples of this we saw in past years were "final reports" that made it easy to find the code, summarized the current state of the project, and enumerated challenges and learnings