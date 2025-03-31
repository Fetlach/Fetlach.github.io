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

<h2>Background</h2>
This project is currently ongoing. I started working on it around July 2024 and have been adding extensions to it periodically alongside my work and academic studies, with brief pauses for side projects such as the tessel8r godray recreation or SDF exploration.

The primary purpose for this project has been groundwork for a larger project I would like to make at a later date. For that future project I would need a robust system for handling runtime procedural generation with aspects not applicable to voxel or hexgrid implementations. This in of itself is a large challenge, but after breaking it down into a series of smaller projects it has been a fun passtime to tackle.

<h2>Overview</h2>

The project is an unreal engine plugin that currently has functionality for building voronoi graphs, procedural meshes, hierarchical scene graphs, serialization/deserialization, and pathfinding all at runtime with the ability to use these features in blueprints. These features are purposed for terrain generation and usage specifically.

The plugin currently is composed of C++ function libraries and UE classes which can be used in the editor, with plans for an editor mode that makes editing terrain generation with the plugin easier.

There are also plans to revise and revisit portions of the codebase. This is to better integrate with UE blueprints and to abstract away lower-level parts of the project for easier usage without prior understanding.

<h2>Why use voronoi?</h2>

It's a tough question to answer empirically; Voronoi doesn't offer any particular mechanical benefits over square, hexagonal, or triangular grids. Technically one could argue that voronoi doesn't have as much of an issue with prominent-axis movement, but it's not very convincing. The best reasons my brother and I could come up with were the ability to expand and contract tile spacings to intuitively show movement cost and having a variable number of neighboring tiles to spice up tile-based tactics.

I decided that "it looks cool" is a good enough reason in of itself.<br>
If that's too much of a cop-out, there's also loads of cool tile arrangements that can be elegantly expressed with a voronoi graph that can't be easily made otherwise. Voronoi graphs can be made to mimic many regular tilings so long as each tile is convex, so there's nothing stopping you from merging a hex grid into a rectanglular one or tweaking the tile positions slightly on a chessboard to mess with people.

<h2>The voronoi implementation</h2>

So let's get started!
I recommend finding a library to handle voronoi graph generation from a pointcloud since we'll need to roll our own code elsewhere and implementations for voronoi are common enough already, but I'll still talk about the implementation anyway.

Voronoi graph generation is typically done using Fortune's Algorithm with an underlying half-edge data structure to store the graph. Broadly, it works by performing a sweep across a 2D point cloud and maintains a "beach line" to track parabolic intersections as they occur. This quickly finds the intersections between the cells and can be modified to use other spatial expressions such as manhattan or parabolic distance by changing how the beach line is constructed.

However, this alone is not enough to complete the graph as you'll end up with infinite edges afterwards. The best approach to handle this is to bound the graph to a shape like a square or triangle. This leaves you with a complete finite voronoi tesselation of a shape, and is extendable to any number of dimensions you desire.

<h2>The geometry builder</h2>

Once you have a voronoi graph, it's time to give it a visual representation!

There's many approaches depending on the exact use case, but a commonality between them is the use of a polygonal triangulation function. Since every tile is guaranteed to be convex (in euclidean space), it is sufficient to use a function to arrange the points in a given winding (if they are not already) and follow with the naive fan-folding algorithm to create a triangulation.
These triangles can be passed to a procedural geometry implementation of your choice, and serve as the biggest hurdle to more complex generation.

After getting the faces and their triangulations, the rest of whatever you want to do for the tile's geometry is fairly simple so long as you think it through. The approach I had the best success with was by walking the face according to it's vertex winding, as it allows you to enumerate each face fairly consistently.

UV coordinates can be programmatically determined by setting an anchor on a face, and walking accordingly. You can unwrap with seams along each vertical edge, or split along the face depending on what sort of style you're going for. Each has its merits, but be aware that both can look bad if you're not careful. I suggest adding another simple geometry element to cover the UV seams and accuentuate the look.

In my Unreal implementation, I sped up the terrain generation by creating a geometry factory that appends a tile's desired geometry to a large list which is passed all at once to the procedural mesh component. 

There are a few benefits to using voronoi here that are not immediately obvious. The first is that the resulting polygon counts will be relatively low depsite the forms since even with more complex generation we're fitting our forms pretty closely. The second is that collision convexes are extrordinarily easy to approximate for most implementations which can speed up generation time massively if you create them yourself. These can both be leveraged to generate complex-looking geometry at blazingly fast speeds even at runtime on current-day lower-end hardware.

<h2>SDF trees & variable Poisson sampling</h2>

As an added benefit, it is possible to fill up areas in a way that is more aesthetically pleasing than simple noise. And as an extension, we can open this process up with a high degree of creative control.

The first extension is using variable density Poisson-disc sampling to generate or fill in pointclouds that can be pushed into our voronoi graph generator. Poisson sampling is often used in dithering to generate a distribution of points that grows more condensed or sparse depending on a provided density function. The density function can be anything so long as it can produce an output for any given location which makes input such as images possible.

This is where SDF trees come into play. Signed Distance Field functions can be used to test if various points in space are inside, outside, or on the edge of the shape they represent. SDF functions are continuous, math-based, and can be combined which gives us the ability to chain them together into trees.

<h2>Pathfinding</h2>

While navigation might seem intimidating based on everything we've seen so far, it's actually one of the more straightforward hurdles. Graph-based pathfinding is a well-documented computational problem, and any sort of A Star implementation will work to navigate the voronoi graph. Since I built this project in unreal, I used the GraphAStar implementation.

However, voronoi graphs tend to have issues with representing empty space. While it's alright to save "air" tiles here and there, they can quickly spiral out of control or have adverse affects on the terrain geometry depending on implementation. So what's to be done?

<h2>The hierarchal graph<h2>

This also sounds intimidating, but it's not that bad either.<br>
The gist is to "chunk" the voronoi graphs by breaking them up into a set of hierarchically organized doubly-connected graphs.
I found that organizing the graphs into three tiers makes the most sense:<br>
<ul>
<li>Tier 0: Map Sections, which represent fully-connected islands on a pointmap</li>
<li>Tier 1: Maps, which represents a single pointmap each</li>
<li>Tier 2: Hyper Maps, a single parent of all maps in the scene</li>
</ul>
There also need to be some rules imposed to make this system work;
<ul>
<li>Graphs should only be connected bidirectionally to graphs in the same tier.</li>
<li>Graphs should only have children or parent of one tier lower or higher respectively</li>
<li>No graph of any tier should share references with its siblings on the same tier</li>
<li>Only Tier 0 graphs should be used for per-tile pathfinding</li>
<li>Only Tier 1 graphs should house references to the pointmaps</li>
<li></li>
</ul>
While this implementation does complicate pathfinding, it also allows for faster pathfinding in general on well-defined or sparse terrain by leveraging a quick planning pass to greatly narrow down which tiles to search through before performing the more expensive tile-by-tile pathfinding.

Since this system is also totally abstract, it supports many fun extensions out-of-the-box such as non-euclidean and dynamic connections. For instance, you could have a graph of tiles move around and "dock" with other islands by just adding and dropping connections to the other graphs.

<h2>Current work-in-progress: Saving and Loading</h2>

Due to the complexity of the data being used, saving and loading from a file is an obvious next step. Picking a good format required considering a few criteria: 
<ul>
<li>The data itself is too bulky for basic unreal engine serialization, which uses JSON formatting.</li>
<li>While human-readible data is often beneficial, the graph format is difficult to make sense of from pure data alone.</li>
<li>Files should ideally be small to avoid memory creep and overhead on file sharing via source control.</li>
<li>The serialization setup should ideally minimize any impact from save compression on the data.</li>
<li>Transferability is a plus; level editing is a future goal, which may desire an external editor.</li>
<li>High serialization/deserialization speeds are a plus: saving and loading quickly is always good.</li>
</ul>
After a bit of reasearch, I concluded that using Google's protocol buffers would be a good solution to most of these criteria. As an overview, it is a language and platform neutral serialization method developed for data exchange between different systems. While it would take a bit more setup than other alternatives, protobuf (protocol buffers) offers many advantages compared to other available serialization formats and will be extensible for later portions of the project. 

To function, protobuf requires a data schema to serialize and deserialize correctly. Here's the setup I started with;

[TODO: insert schema here]

It is difficult to assign object pointer references within the schema without causing issues, so an ID is assigned to most objects during serialization. These IDs can be used during deserialization with an array of pointer redirectors to easily recover pointer-based connections.
Another non-obvious optimization is to pool the voronoi graph's vertices into a list of local 2D vectors and keep a separate transform representation for the tiles and graph. This significantly culls the number of floats stored and allows us to store the vertices on a normalized 0.0 to 1.0 axis setup for greater floating-point precision. 