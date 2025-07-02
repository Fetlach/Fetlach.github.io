---
layout: post
title:  "Meltygrid Project"
summary: "going off-grid with voronoi"
author: fetlach
date: '2099-03-24'
category: ['wip']
tags: project
thumbnail: /assets/img/post_neography/VagaboundND.png
keywords: Voronoi, Unreal Engine, Plugin, Computational Geometry
usemathjax: false
permalink: /blog/meltygrid-3-24-2025/
---

<h2>Overview</h2>

This project is an unreal engine plugin that looks to provide a flexible alternative to regular grids while still keeping their main benefits.

<h2>The Main Idea</h2>

At its core the project uses and builds upon a half-edge data structure to generalize planar faces and face adjacency, not unlike traditional 3D modeling applications. Though, this generalized representation provides a huge degree of flexibility by interfacing with other systems.

The plugin currently is composed of C++ function libraries and UE classes which can be used in the editor, with many plans being worked on for extending its capabilities and allowing easier editing.

There are also plans to revise and revisit portions of the codebase. This is to better integrate with UE blueprints and to abstract away lower-level parts of the project for easier usage without prior understanding.

<h2> Building Blocks</h2>

<h2>Why Use Voronoi?</h2>

It's a tough question to answer empirically; Voronoi doesn't offer many particular mechanical benefits over square, hexagonal, or triangular grids. 
Even after racking our brains, my brother and I struggled to think of benefits over existing systems in current gameplay genres that could justify its implementation cost for most users.

However in the context of this project the usage of Voronoi offers many key advantages and areas that could prove interesting:
<ul> 
<li>Voronoi diagrams can use KNN to partition space; Finding the region at a given coordinate is the same as a Nearest-Neighbor query</li>
<li>The creation of a quadtree accelerates 2D KNN queries to a time complexity of O(log(n)) - fast enough for runtime usage</li>
<li>LLoyd relaxation can be used to evenly spread regions across a space</li>
<li>Sites can be arranged to recreate and blend between certain regular minimal tilings</li>
<li>The dual graph of a Voronoi diagram is a Delaunay triangulation of its sites</li>
<li>The diagrams can be generated quickly from a set of sites in O(n * log(n)) time complexity with Fortune's algorithm</li>
</ul>

Additionally, one of the nice parts about Voronoi is that you don't have to lose out on the main benefits of a grid by creating a subgrid within the diagram.
Square, Hexagonal, Triangular, and many other tilings can be constructed within a Voronoi diagram by positioning the sites correctly and adding a small buffer around that space.
This allows most types of modular assets to be freely spawned in that area without interference, and opens the possibility of rotating the placement of those structures.
Adding a mapping system to that area even allows retaining the fast tile lookup speeds.
Aside from the difficulty with implementation and editing, it can be a direct improvement on most aspects of the grids we know.

<h2>The Asset Pipeline</h2>

But this idea is only so useful as it can be made to work - So a system needs to be built on top of our diagrams to generate geometry.

Luckily, we have tools at our disposal to complete this process efficiently:
<ul>
<li>Polygon triangulation algorithms (Fan-folding, Ear-clipping)</li>
<li>Procedural meshes</li>
<li>2D normal precomputation for diagram edges</li>
</ul>

While there's many approaches for how the mesh is going to be generated for a region depending on the exact use case, a commonality between them is the use of a polygonal triangulation function. So long as we can guarantee a simple polygon without holes as input we can use an O(n^2) ear-clipping algorithm to triangulate the polygon, and we can speed this up to O(n) with the fan-folding algorithm if we know ahead of time that the polygon is simple and convex.

From there, some manner of extrusion is needed to bring the region into 3D space. While more complex operations such as beveling may add complexity, the overall result can be very performant as many features such as normals can be determined before even considering 3D space.

One of the notable remaining bottlenecks in procedural mesh generation is the creation of collision. And it's a doozy.
While we could use our output meshes as complex geometry and incurr a hefty performance hit, we don't have to! Going back to our tools; The abiliy to decompose the 2D diagram via triangulation also allows us to greedily combine the triangles of a face to a 2D convex. Extrusion of these 2D convexes along a third axis will yield 3D convexes, which are exactly what we would need for collision and is one of the faster collision types to use.

Other than the above, only the UVs are dependant on approach. 
They can be assigned any number of ways to apply textures so there is not a fantastic generalizable system, especially when more complex geometry is used and we can no longer rely on prism-based arrangements.
UV coordinates can be programmatically determined by setting an anchor on a face, and walking the features generated from each edge according to the 2D vertex winding. You can unwrap with seams along each vertical edge, or split along the face depending on what sort of style you're going for. Each has its merits, but be aware that both can look bad if you're not careful. I suggest sidestepping a lot of this process by adding another simple geometry element to cover the UV seams and accuentuate the look.

All of this can be leveraged to generate complex-looking geometry at blazingly fast speeds even at runtime on current-day lower-end hardware. The results aren't usually too costly on the rendering side either because the outputs are generally on the lower side in poly count and the lighting can often be faked as well.

<h2>Pointcloud Editing, SDF Trees, and Variable Poisson Sampling</h2>

As an added benefit, it is possible to fill up areas in a way that is more aesthetically pleasing than simple noise. And as an extension, we can open this process up with a high degree of creative control.

The first extension is using variable density Poisson-disc sampling to generate or fill in pointclouds that can be pushed into our Voronoi generator. Poisson sampling is often used in dithering to generate a distribution of points that grows more condensed or sparse depending on a provided density function. The density function can be anything so long as it can produce an output for any given location which makes input such as images possible.

This is where SDF trees come into play. Signed Distance Field functions can be used to test if various points in space are inside, outside, or on the edge of the shape they represent. SDF functions are continuous, math-based, and can be combined which gives us the ability to chain them together into trees. So if you have a little processing power available to sample them it's quite easy to throw around a couple SDFs and generate a pointcloud.

And of course, after a pointcloud is generated then all sorts of extensions could be added to make editing a breeze. Just adjust the positions of the sites, and regenerate that area of the diagram.

Most of these things can also be automated to a hilarious degree. Given, you'll need to think a bit on how the procedural generation should work when the space is less discretized than normal.

<h2>The Hierarchal Graph<h2>

So where do we go from here?

One aspect that grids have over this system so far is that they can be "chunked". It saves on memory and rendering by having each chunk be loaded in only when you need it.
But we can tackle that too, and do it in a highly flexible way.

The gist is to "chunk" the voronoi graphs by breaking them up into a tree of hierarchically organized graphs with bidirectionally connected leaves.
That's a mouthful, but here's a breakdown:
<ul>
<li>Tier 0: Map Sections, which represent fully-connected islands on a diagram</li>
<li>Tier 1: Maps, which represents a single diagram each</li>
<li>Tier 2: Hyper-Maps, which are used as a single shared parent of all active diagrams in the scene</li>
</ul>

This simple organization is enough to facilitate hierarchical pathfinding and all kinds of dynamic environments. The only cost is the added overhead of determining and marking if areas are traversable via a floodfill algorithm.

While this implementation does complicate pathfinding due to needing a planning pass, it allows for faster pathfinding in general on well-defined or sparse terrain by greatly narrowing down which parts of the diagrams to search through before performing the more expensive tile-by-tile pathfinding. This can even be extended to exact coordinates on a navmesh if you build and cache one during the geometry pipeline from the triangulated faces.

Since this system is also totally abstract, it supports many fun extensions out-of-the-box such as non-euclidean and dynamic connections. For instance, you could have a diagram move around and "dock" with others by just adding and dropping connections.

