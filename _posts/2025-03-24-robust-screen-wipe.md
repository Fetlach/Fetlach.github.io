---
layout: post
title:  "Generalized Screen Wipe"
summary: "Making a more robust screen wipe"
author: fetlach
date: '2099-03-24'
category: ['projects']
tags: projects
thumbnail: /assets/img/post_neography/VagaboundND.png
keywords: UI, UX, Shaders, Unreal Engine, Voronoi
usemathjax: false
permalink: /blog/screen-wipe-3-24-2025/
---

<h2>Overview</h2>

A VFX that has often annoyed me is the well-known screen wipe; It's well documented and easy to implement right up until you try to customize it.
Vertical and horizontal wipes? No problem - You could probably find a version for your tool within minutes.
However a generalizable wipe that works in any direction is a little harder to come across since there's some math involved - Which I'll outline here in this project breakdown!

<h2>What to Consider</h2>

The main challenges we need to tackle here are a variable aspect ratio and having the wipe support gradients.

The first part is self-explanitory; A non-orthogonal wipe without correction will be skewed on anything but a 1:1 AR, and skews dependant on how far away from 1:1 the AR gets.
That's bad because generally designers would like their elements to read the same on their standard 1920x1080 preview and a user's ultrawide monitor.

The second part is a usability problem - While we could treshold to solve the first challenge it wouldn't look very good, and there's many scenarios where having a soft edge on the wipe might be preferable.

<h2>The Math</h2>