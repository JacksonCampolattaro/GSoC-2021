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
We decided that if we wanted to take advantage of SIMD we would need to find places exactness wasn't necesary.

### N-Way Tree

### Experimenting with Workarounds

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