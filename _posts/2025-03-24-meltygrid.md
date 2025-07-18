---
layout: post
title:  "Meltygrid Project"
summary: "Going off-grid with Voronoi"
author: fetlach
date: '2025-07-02'
category: ['projects']
tags: project
thumbnail: /assets/img/posts/Meltygrid/Meltygrid.png
keywords: Voronoi, Unreal Engine, Plugin, Computational Geometry
usemathjax: false
permalink: /blog/meltygrid-3-24-2025/
featured: true
---

<h2>Overview</h2>

This project is an unreal engine plugin I have been developing that looks to provide a flexible alternative to regular grids while still keeping their main benefits.
It's still WIP as I need to flesh out the prototype and finish implementation of core features, but shows enough promise that I wanted to talk about the progress so far.

The idea for this project came from studio Ludomotion's work on [Unexplored 2](https://store.steampowered.com/app/1095040/Unexplored_2_The_Wayfarers_Legacy/), where I believe they were onto something big. While Ludomotion was more interested in exploring the topic from the perspective of using it as a medium to implement the [cyclical generation](https://www.boristhebrave.com/2021/04/10/dungeon-generation-in-unexplored/) of [Unexplored 2](https://store.steampowered.com/app/506870/Unexplored/) , they were ultimately limited by an underlying grid and thus were unable to make use of what I believe are the cooler applications of the tech.

At its core the project uses and builds upon a graph structure to generalize planar faces and face adjacency, not unlike traditional 3D modeling applications. Though, this generalization provides a huge degree of flexibility as an interface with other systems. For the moment this includes a procedural meshing pipeline, graph traversal schemes, and a framework for sample-based procedural generation. Though there are plans to finish and extend many aspects of the project in the near future.

The plugin currently is composed of C++ function libraries and UE classes which can be used in the editor, with many plans being worked on for extending its capabilities and allowing easier editing.

There are also plans to revise and revisit portions of the codebase. This is to better integrate with UE blueprints and to abstract away lower-level parts of the project for easier usage without prior understanding.

<h2>Building Blocks</h2>

The interesting part about this project is how most of the tech used has been around for a while. Though, the combination of these smaller parts has gone largely unexplored thanks to the tremendous time and effort needed to research, design, and implement a cohesive system which makes use of their qualities.

Currently, the following pieces of tech are being used:

* [Half-Edge planar diagrams](https://jerryyin.info/geometry-processing-algorithms/half-edge/)
* [Voronoi diagrams](https://www.redblobgames.com/x/2022-voronoi-maps-tutorial/)
* Quadtree KNN querying
* SDF functions and SDF trees
* A* pathfinding and graph traversal
* Hierarchal graphs
* Procedural mesh pipeline


And there are also plans to incorperate the following pieces of tech in the future:

* [Protobuf serialization & deserialization](https://github.com/protocolbuffers/protobuf)
* Polygonal clipping
* Support for concave polygons
* Supporting function library


<h2>Why Use Voronoi?</h2>

The most noteworthy part about this project, and why it might be generally useful, is the ability to leverage the full strengths of Voronoi diagrams. 
Voronoi diagrams are aesthetically pleasing being able to resemble natural patterns and certain tilings. 
Though what benefits beyond that would you get?

It's a tough question to answer empirically; Voronoi offers only a few particular mechanical benefits over square, hexagonal, or triangular grids. 
Even after racking our brains, my brother and I struggled to think of benefits over existing systems in current gameplay genres that could justify the upfront implementation cost for most users. 
This is the main reason why Voronoi hasn't been explored in the gaming industry much beyond usage in noise materials.

However in the context of this project Voronoi offers many key advantages:

* Voronoi diagrams can use KNN to partition space; Finding the region at a given coordinate is the same as a Nearest-Neighbor query
* The creation of a quadtree accelerates KNN queries to a time complexity of O(log(n))
* LLoyd relaxation can be used to evenly spread regions across a space
* Sites can be arranged to recreate and blend between certain regular minimal tilings
* The dual graph of a Voronoi diagram is a Delaunay triangulation of its sites
* The diagrams can be generated quickly from a set of sites in O(n * log(n)) time complexity with Fortune's algorithm


Additionally, one of the nice parts about Voronoi is that you don't have to lose out on the main benefits of a grid by creating a subgrid within the diagram.
Square, Hexagonal, Triangular, and many other tilings can be constructed within a Voronoi diagram by positioning the sites correctly and adding a small buffer around that space.

This allows most types of modular assets to be spawned in a regular area without interference by an anotherwise irregular diagram, and opens the possibility of rotating the placement of those structures or blending between different regular grids to facilitate asset arrangements.

Adding a mapping system to a regular area even allows retaining the fast tile lookup speeds, so aside from the upfront difficulty with implementation which this project addresses it can be a direct improvement on most aspects of the grids we know.

<h2>The Asset Pipeline</h2>

But this idea is only so useful as it can be made to work - So a system needs to be built on top of our diagrams to generate geometry.
Since we can't rely on conventional methods such as mesh instancing and path tracing can be slow, we'll need to explore how to generate geometry that is compliant to the underlying structure.

Luckily, we have tools at our disposal to complete this process efficiently:

* [Polygon triangulation algorithms](https://en.wikipedia.org/wiki/Polygon_triangulation/)
* Procedural meshes
* 2D normal precomputation for diagram edges

While there's many approaches for how the mesh is going to be generated for a region depending on the exact use case, a commonality between them is the use of a polygonal triangulation function. So long as we can guarantee a simple polygon without holes as input we can use an O(n^2) ear-clipping algorithm to triangulate the polygon, and we can speed this up to O(n) with the fan-folding algorithm if we know ahead of time that the polygon is simple and convex.

From there, some manner of extrusion is needed to bring the region into 3D space. While more complex operations such as beveling may add complexity, the overall result can be very performant as many features such as normals can be determined before even considering 3D space.

<center><img src="/assets/img/posts/Meltygrid/BigGraph.png" class="img-fluid"></center><br>

One of the notable remaining bottlenecks in procedural mesh generation is the creation of collision. And it's a doozy.
While we could use our output meshes as complex geometry and incur a hefty performance hit, we don't have to! Going back to our tools; The abiliy to decompose the 2D diagram via triangulation also allows us to greedily combine the triangles of a face to a 2D convex. Extrusion of these 2D convexes along a third axis will yield 3D convexes, which are exactly what we would need for collision and is one of the faster collision types to use.

Other than the above, only the UVs are dependant on approach. 
I haven't come up with a generic solution for generating the UVs in a way that can automatically apply a texture, though it may be possible with a pipeline that assumes more constraints. I suggest sidestepping a lot of this part by using non-UV dependant materials or adding another simple geometry element to cover the UV seams. Especially in the case you would like to have something to indicate the center of a tile.

All of this can be leveraged to generate complex-looking geometry at blazingly fast speeds even at runtime on current-day lower-end hardware. The results aren't usually too costly on the rendering side either because we aren't sampling a discretized version of the diagram unlike traditional heightmap implementations.

<h2>Pointcloud Editing, SDF Trees, and Variable Poisson Sampling</h2>

As an added benefit, it is possible to fill up areas in a way that is more aesthetically pleasing than simple noise or adding noise to an underlying grid. 
And as an extension, we can open this process up with a high degree of creative control.

The first extension is using variable density [Poisson-disc sampling](https://en.wikipedia.org/wiki/Supersampling#Poisson_disk) to generate or fill in pointclouds that can be pushed into our Voronoi generator. Poisson sampling is often used in dithering to generate a distribution of points that grows more condensed or sparse depending on a provided density function. The density function can be anything so long as it can produce an output for any given location which makes input such as images possible.

This is where SDF trees come into play. [Signed Distance Field functions](https://iquilezles.org/articles/distfunctions/) can be used to test if various points in space are inside, outside, or on the edge of the shape they represent. SDF functions are continuous, math-based, and can be combined which gives us the ability to chain them together into trees. So if you have a little processing power available to sample them it's quite easy to throw around a couple SDFs and generate a pointcloud to be used as input for our diagram like so:

<center><img src="/assets/img/posts/Meltygrid/VTVisualizer.png" class="img-fluid"></center><br>

And of course, after a pointcloud is generated editing a breeze. Just adjust the positions of the sites, and regenerate that area of the diagram.

Most of this part can be automated to a hilarious degree. Given, you'll need to think about how the procedural generation should work when the space is less discretized than normal.

<h2>The Hierarchal Graph</h2>

So where do we go from here?

One aspect that grids have over this system so far is that they can be "chunked". It saves on memory and rendering by having each chunk be loaded in only when you need it.
But we can tackle that too, and do it in a highly flexible way.

The gist is to "chunk" the voronoi graphs by breaking them up into a tree of hierarchically organized graphs with bidirectionally connected leaves.
That's a mouthful, but here's a breakdown:
<ul>
<li>Level 0: Root, which are used as a single shared parent of all active diagrams in the scene</li>
<li>Level 1: Maps, which represent a single diagram each</li>
<li>Level 2: Map Sections, which represent fully-connected islands on a diagram</li>
</ul>

<center><img src="/assets/img/posts/Meltygrid/MGHGraph.png" class="img-fluid" width="500"></center><br>

Level 0 nodes are parents of Level 1 nodes, and the same applies for Level 1 and Level 2.
This simple organization scheme is enough to facilitate hierarchical pathfinding and all kinds of dynamic environments. The tradeoff is a small added overhead of determining and marking if areas are traversable via a [floodfill algorithm](https://en.wikipedia.org/wiki/Breadth-first_search).

While this implementation does complicate pathfinding due to needing a planning pass, it allows for faster pathfinding in general on well-defined or sparse terrain by greatly narrowing down which parts of the diagrams to search through before performing the more expensive tile-by-tile pathfinding. This can even be extended to exact coordinates on a navmesh if you build and cache one during the geometry pipeline from the triangulated faces.

Since this system is also totally abstract, it supports many fun extensions out-of-the-box such as non-euclidean and dynamic connections. For instance, you could have a diagram move around and "dock" with others by just adding and dropping connections.

<h2>Notes On Procedural Generation</h2>

The implications of this tech on procedural generation are similarly exciting.

As mentioned previously, the mesh generation pipeline is capable of outputting representations of the underlying data structure. However, these aspects are not coupled;
A dedicated team can use the pipeline to generate and export a greyboxed terrain for reference and then create a static higher-quality environment manually in an external tool. So long as the new environment doesn't deviate too much from the original diagram, the diagram's component can use that mesh instead without any downsides. 
This has some major ramifications for procedural generation as prefabs could have far fewer layout restrictions and use 2D polygonal bounds. It also allows for the possibility of automatically filling in spaces between prefabs with sites using "skirt" bounds.

Graph searches can find the extents of certain regions or provide general methods of spawning objects in a loop or line on a diagram. For Instance a routine using BFS and a radial sort to spawn legs along the outside of a diagram.

<center><img src="/assets/img/posts/Meltygrid/leg placement.png" class="img-fluid"></center><br>

Unfortunately, a limitation with the current system is updating generated meshes at runtime for irregular tile arrangements. 
While systems that use instancing such as voxel or tile based systems can do this much easier, it is not so easy to update procedural mesh sections without incurring a hit to performance. This can be partially worked around by breaking the extrusions down into walls and faces such that walls could have their scalings updated or be hidden. Though, we cannot pre-generate a mesh for N-sided polygons and workarounds for this are not very feasible for general use. 
However, full optimization via ISM can be achieved for [regular tilings](https://en.wikipedia.org/wiki/Euclidean_tilings_by_convex_regular_polygons#Archimedean,_uniform_or_semiregular_tilings) by pre-generating a small set of meshes for the top faces so long as the tiles can each be represented by a single site (or decomposed into multiple tiles for more complex arrangements).