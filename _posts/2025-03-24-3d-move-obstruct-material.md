---
layout: post
title:  "Stylized Debris Obstruction"
summary: "Getting out of view with WPO"
author: fetlach
date: '2025-07-09'
category: ['projects']
tags: projects
thumbnail: /assets/img/posts/WPO/WPO_thumbnail.PNG
keywords: UX, Shaders, Unreal Engine, Materials, VFX
usemathjax: false
permalink: /blog/wpo-obstruction-3-24-2025/
---

<h2>Overview</h2>

During my undergrad I did a playthrough of divinity: original sin. Fun game.
Though, one thing that began bugging me was how the occlusion effect was the same as every other game out there.

Previously for [the first studio class project in my undergrad](https://drive.google.com/file/d/13wvs8f6vClH0KVTyFoU5-MoB6th2asxc/view?usp=sharing) my team and I created a pacman-like game about a cat running around in a crypt.
One of the parts of the aptly named "Cat-a-combs" I was most proud of were the walls of the maze that would drop out of the way when the player was behind them - An idea we got from a few JRPG indie games online.
However, being my first full project the implementation was not very good. The effect needed micromanaging using colliders and there was A LOT of ticking involved.

So this project was about how to make the occlusion effect more immesive, more performant, and fire-and-forget for convenience.

<h2>Showcase</h2>

<center><iframe width="560" height="315" src="https://www.youtube.com/embed/js4t32yRIOs?si=0htqylErrRq1ltxG" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></center><br>

<h2>Side Notes and Limitations</h2>

About a year or two after I did this project Animal Well released with a similar VFX for one of the later areas. 
That implementaiton was for a 2D context, but it looks really good! 
I bring this up because the basic 3D version will look very similar in practice. This is because the "push" vector I describe for the obejcts is normally close to or perpendicular to the camera's view, and so the push will appear to be 2D to the camera.

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

```c
// Point D is the closest point to C on our line, so we need to do a projection of C onto AB
Vector3D LineDirection = normalize(B - A);
float dotValue = dot(C, LineDirection);
Vector3D D = (dotValue * LineDirection) + A;
```

Then, the normalized Vector DC is the perpendicular direction vector we'd like to push the object in. Easy!

```c
// The vector D -> C is the push direction
Vector3D PushVector = C - D;

// We can also break the push vector into a direction and magnitude
float pushScalar = length(PushVector);
Vector3D pushDirection = normalize(PushVector);
```

The second part, how much to push the object, depends on what you want to achieve with the effect.
An easy version would be to push according to the inverse of the distance from the line:

```c
// We'll say the distance we want to push to is X
float distanceThreshold = X;
float pushAmount = clamp((1.0 - (pushScalar / distanceThreshold)), 0.0, 1.0);
```

Though there is a far better option for customization:
For my showcase I've used a 3D capsule SDF as an example of an "activator". You can check out a list of SDF implementations [here](https://iquilezles.org/articles/distfunctions/) - Some other good ideas are a rounded cone or a polygonal extrusion.

The idea here is to ease into the effect with a smoothstep on distance thresholds like above. We return a 1 when the point is on or inside the SDF, and smoothly interpolate to 0 as our query point reaches a certain distance away from the SDF's border. This lets us smoothly interpolate the push in and out automatically as our input vectors move around the scene.

<h2>Variations</h2>

So now that we have the building blocks for this type of effect, what can we do with them?

As shown in the video; we can use the push direction vector, push direction scalar, and activation strength scalar.
The two scalar outputs can drive material lerps such as changes in color strength, and the push direction can of course be used to push objects.
These properties can be used alongside simple noise to randomize properties of the effects per-object as well.

The combos I found that were easy to implement and good-looking combined the push with a secondary action such as a scaling, opacity masking, or rotation. 

Rotation is a little tougher, though looks the best, as the axes need to be defined in 3d. 
There's two main approaches - either using a worldspace axis (such as in my planks example) or matching a local rotation. While you can get away with using a worldspace axis, it quickly becomes difficult to utilize with mesh instancing as for most scenarios you will want more than one rotation axis across your instances and randomizing according to noise doesn't always work.
The better general-use option is to declare a direction on the object that you would like to rotate towards the projected point D we calculated earlier. This can be accomplished through a ["find look at rotation"](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/Math/Rotator/FindLookatRotation) implementation using the object's location and [blending the rotation](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/Math/Rotator/Lerp_Rotator) accordingly.