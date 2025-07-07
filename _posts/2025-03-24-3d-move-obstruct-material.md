---
layout: post
title:  "Stylized 3D Obstruction Materal"
summary: "Getting out of view with WPO"
author: fetlach
date: '2099-03-24'
category: ['wip']
tags: project
thumbnail: /assets/img/post_neography/VagaboundND.png
keywords: UX, Shaders, Unreal Engine, Materials, VFX
usemathjax: false
permalink: /blog/wpo-obstruction-3-24-2025/
---

<h2>Overview</h2>

During my undergrad I did a playthrough of divinity: original sin. Fun game.
Though, one thing that began bugging me was how the occlusion effect was the same as every other game out there.

Previously for the first studio class project in my undergrad my team and I created a pacman-like game about a cat running around in a crypt.
One of the parts of the aptly named "Cat-a-combs" I was most proud of were the walls of the maze that would drop out of the way when the player was behind them - An idea we got from a few JRPG indie games online.
However, being my first full project the implementation was not very good. The effect needed micromanaging and there was A LOT of ticking involved.

So this project was about how to make the occlusion effect more immesive, more performant, and fire-and-forget for convenience.

<h2>Side Notes and Limitations</h2>

About a year or two after I did this project Animal Well released with a similar VFX for one of the later areas. 
That implementaiton was for a 2D context, but it looks really good! 
I bring this up because the basic 3D version will look very similar in practice. This is because the "push" vector I describe for the obejcts is close to or perpendicular to the camera's view, and so will generally appear to be 2D to the camera.

However, this VFX does have its limitations. Being a vertex-based effect it will not be able to function well with larger objects, so we're limited to smaller debris or chunked materials like wood panelling. 
The effect can be modified to use the centroid of a mesh's tris instead of the object's pivot to work with higher polycount meshes by pushing individual tris around, but doesn't usually fit neatly into standard asset pipelines as it requires unique vertices per tri.
So it's best to combine this VFX with a standard dissolve occlusion shader - Use this one for smaller objects, and a dissolve occlusion shader for larger objects.

Another thing to note is while WPO objects can recieve shadows, the shadowmass they contribute to the scene doesn't move with them. For smaller objects it won't usually make a difference on the surrounding area if they contribute to shadows or not, so be sure to turn that off to avoid self-shadowing artifacts during the effect.

<h2>The Math</h2>

The core vector math needed to achieve this effect is very simple and lightweight, though we'll need to add some other parts to make it look good automatically.

We have three main variables to consider: Our camera's position (Vector A), a position in the scene (Vector B), and our object's pivot (Vector C).
And the math is divided into two parts: Determining the direction to push the object, and how much to push the object.

The first part, finding the direction to push the object, reduces to finding the closest point (Vector D) on our Line AB to our Vector C.
The math is simple:

Then, the normalized Vector DC is the perpendicular direction vector we'd like to push the object in. Easy!

The second part, how much to push the object, depends on what you want to achieve with the effect.
For this tutorial I'll use a 3D capsule SDF as an example of an "activator". You can check out a list of SDF implementations [here](https://iquilezles.org/articles/distfunctions/) - Some other good ideas are a rounded cone or a polygonal extrusion.

The idea here is to ease into the effect with a smoothstep on distance thresholds. We return a 1 when the point is on or inside the SDF, and smoothly interpolate to 0 as our query point gets further away. This lets us smoothly interpolate the push in and out automatically as our input vectors move around the scene.