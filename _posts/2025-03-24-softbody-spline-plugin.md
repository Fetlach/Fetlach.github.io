---
layout: post
title:  "XPBD Softbody Spline Plugin"
summary: "My Texas A&M undergraduate capstone project"
author: fetlach
date: '2024-06-01'
category: ['projects']
tags: project
thumbnail: /assets/img/post_neography/VagaboundND.png
keywords: Unreal Engine, Plugin, Physics, XPBD, Rendering, VFX
usemathjax: false
permalink: /blog/xpbd-whip-3-24-2025/
---

<h2> Background </h2>

In the visualization program at Texas A&M we are required to complete an undergraduate capstone project as part of our coursework.
After doing some market research, I found that as of January 2024 there was a distinct lack of game-ready softbody weapons or plugins to make them available on the major digital marketplaces.

Softbody-Spline-based implementations such as flails or whips in particular were lacking. Most assets available to purchase used either animations or basic rope-style physics.
While these approaches can work in certain contexts, they don't generally look great. 
Animation-based approaches are slow to implement, often need adjustment, and are hard to have interact with an environment. 
Rope-style physics approaches are quick to implement and interact with the environment, but don't behave as expected due to uniform mass and collision size.

This proposal was accepted by professor Kicklighter, and he did a great job at keeping me on track during the three months of R&D while learning how to implement physics.
Over the course of this project I was also mentored by Greg Street who taught me to a lot about how to identify what a user would want from a product and usability perspective.

<h2> Extended Particle-Based Dynamics (XPBD) </h2>

The main reason why rope-based physics is often used is because the main method, verlet integration, is very easy to implement and is very stable. 
There is a large gap in implementation needed to have a more robust system, and thus it isn't usually in the interests of a developer to do the research and implementation themselves.

Luckily for us, a research team at Nvidia produced a great paper outlining a particle-based physics solver that fits exactly what we're looking for and is not too bad of a step up. 
XPBD, or "extended particle-based dynamics", is a great extension to prior solvers that vastly improves stability and speed. 
Since it's a solver framework and not an algorithm, we can also implement physics constraints. These allow us to implement variable mass on our particles instead of having everything be uniform and opens the door to other applications.

For the project, I opted to use N-jumping distance constraints for the particles where each particle is created with a distance constraint to the N particles in front of it. 
A rope-based approach is analgous to a N=1 version of this format, as in each particle is only connected to its neighbors. I found N=2 and N=3 with diminishing constraint strength was enough to achieve an accurate result

My implementation of the solver was done in C++ and built around integration with the Unreal Engine framework, though was faithful to the original aside from small fixes for the realtime environment described in later sections.

<h2> Setting Up UE World Collisions </h2>

To have the spline interact with the world, we'll need to add functionality such that the UE physics system can have our component produce and recieve collisions.
This part of the code is largely boilerplate as many standard components use the same functionality. However it is worth noting that we override certain parts to manipulate friction and need to apply collisions across each of our simulated particles instead of one hardbody.

Afterwards, the component can collide with the environment. Awesome!

<h2> Correcting Physics for Realtime </h2>

One aspect of physics solvers I discovered after the initial implementation was that in realtime environments the physics can be wildly impacted by irregular timesteps.
This was solved by instead using a substepped approach where the component might perform multiple timesteps of a regular interval to catch up to the present, or not perform a step at all if the accumulated time isn't great enough.
This method, called "timestep fixing", works very well. Though, a cap on the number of steps we're allowed to perform per-tick is also needed since we don't want the component to lag the game either.

<h2> 3D Splines & Rotation Minimizing Frames </h2>

One challenge in calculating a fitting spline is that if the simulated points were taken as a raw input then the output spline would likely not go through those points.
We can do a bit of additional work to segment the spline into a series of cubic bezier curves where we solve for the tangents at each end and position the bezier anchors accordingly. This produces a series of cubic beziers we can sample N points from to create a spline that looks very accurate to the input. 
This cubic bezier series also serves as the basis for queries in subsequent steps.

However, in 3D a spline's normal and perpendicular are not as well defined.
In 3D a twist will occur in the spline which we would expect the normal to follow. While we can calculate the normals and tangents using just the local position on the spline, when the spline flips concavity so too will these directions flip.
For this reason, we can use [Rotation-Minimizing-Frames](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/Computation-of-rotation-minimizing-frames.pdf) to correct these flips when they occur.

A more robust physics implementation using [cosserat rods](https://www.cosseratrods.org/cosserat_rods/theory/) that was planned but could not be achieved in time sidesteps an issue where RMF may quickly "flip" the end of the spline in certain arrangements. This odd artifact occurs because while RMF is consistent for every arrangement, slightly different permutations might have inverted concavity and yield significantly different results.
Cosserat rods fix this by having several rotational constraints on the particles, and thus inherently keep track of how the spline is rotated at each simulated point.

<h2> Rendering Pipeline </h2>

The component uses a procedural spline mesh that is generated according to editor-exposed variables. 
This implementation is similar to the one outlined in the standard cable component, but makes a few small improvements to customizability described in the next section.
The only drawback here was a problem unaddressed by the Unreal Developers: for softbody components, a transparent material is needed as an opaque material has the velocity buffer enabled by default. 
This causes ghosting artifacts when the scene component moves but the mesh does not move by the same amount - The cable component gets around this by resetting when moved.

<h2> Blueprint Integration </h2>

Contained within the component are several methods for overriding the physics and rendering, and also certain queries for the spline.
Notably, a lot of work was put into getting a lesser-known curve widget working in the editor. These curves were used to define certain physics properties such as mass along the spline, and width profiles for the rendered spline.
This speeds up prototyping dramatically, as certain data that varies across the spline can be easily viewed and manipulated with just a few clicks.

<h2> Sockets and Mesh Attachment </h2>

One of the more interesting features the Unreal Engine offers is the ability to declare sockets on objects. 
While this system can be a bit volatile due to using FNames, we are allowed to override a part of the functionality to update a socket's scene transform. 
Using this I was able to add a small way of declaring where a socket should track to along a spline automatically. 
This interfaces with the rest of the engine's socket implementation nicely, allowing us to attach meshes to the dynamic spline by telling the meshes the name of the corresponding socket.

<h2>Quick Mockup</h2>

After our final report, a friend asked me to do a quick mockup for a weapon from Tokyo Ghoul. 
It was surprisingly easy to implement using the dynamic sockets and ability to modify the shape of the spline - It took about as long to set up as it took to create the meshes.

<center><iframe width="560" height="315" src="https://www.youtube.com/embed/wyW3sSpMyII?si=hpCzSxE3cYvi7xwU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></center>

