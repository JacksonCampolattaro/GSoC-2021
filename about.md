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
- [todo](other reasons)

The AABB-package is only a small part of CGAL,
so a major part of the value of this project is in the institutional knowledge it produced.
As such, I documented my work thoroughly [todo](link to wiki).

> ###:Rubric
> 
> The target of the link should contain: 
> - a short description of what work was done, 
> - what code got merged, 
> - what code didn't get merged, 
> - what's left to do
> 
> The best examples of this we saw in past years were "final reports" that made it easy to find the code, summarized the current state of the project, and enumerated challenges and learnings