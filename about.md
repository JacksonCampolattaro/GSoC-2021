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

Over the course of the project, this goal drifted for several reasons:
- We determined that SIMD was only one part of Embree's performance advantages,
  if we wanted to match their performance we would need to include some other optimizations. 
- The heavily templated & generic nature of CGAL's codebase is not as easily amenable to traditional SIMD techniques 
  as Embree's narrower domain.
- The compiler already does a passable job at vectorizing CGAL's code, 
  which limits how much improvement we can expect from further intentional vectorization.

One of the primary work products that came out of my endeavor this summer was a modification to the 
tree's approach for construction that can increase performance by as much as 50%.
It uses a technique pioneered by Embree, 
building the tree rapidly by sorting the items it contains along a space-filling-curve.
The relevant pull-request is available to view [here](https://github.com/CGAL/cgal/pull/5893),
and documentation for the feature is available [here](https://cgal.github.io/5893/v1/AABB_tree/classCGAL_1_1AABB__traits__construct__by__sorting.html).

The AABB-package ultimately makes up a small fraction of CGAL's code base,
so a major part of the value of this project is in the institutional knowledge it produced.
As such, I documented my work thoroughly on CGAL's internal wiki;
I've created a public mirror of that documentation [here](todo link to wiki mirror).

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

Even more difficult is CGAL's exactness guarantees. 
CGAL can use different math kernels to retain high levels of precision,
and to ensure that rounding errors don't propagate to create incorrect results.
Using SIMD in exact computation could be the subject of its own research project.

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

After further development, I had produced a prototype which fit nicely into CGAL's existing abstraction
and provided several tunable parameters:
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

### Other Techniques used by Embree

### Pull-Request

> ### Rubric
> 
> The target of the link should contain: 
> - a short description of what work was done, 
> - what code got merged, 
> - what code didn't get merged, 
> - what's left to do
> 
> The best examples of this we saw in past years were "final reports" that made it easy to find the code, summarized the current state of the project, and enumerated challenges and learnings