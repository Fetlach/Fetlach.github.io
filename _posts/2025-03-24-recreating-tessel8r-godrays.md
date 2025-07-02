---
layout: post
title:  "Recreating Tessel8r's Godrays"
summary: "a raymarched approach to stylized volumetric lighting"
author: fetlach
date: '2099-03-24'
category: ['projects']
tags: project
thumbnail: /assets/img/post_neography/VagaboundND.png
keywords: Shaders, Unreal Engine, VFX, Raymarching
usemathjax: false
permalink: /blog/tessel8r-godray-3-24-2025/
---

<h2>How it was done originally</h2>

<h2>The problems</h2>

<h2>The solution</h2>

<h2>The math</h2>

<h2>The implementation</h2>

<h2>The results and limitations</h2>
While the results look very nice under certain conditions, there are a few limitations to consider.

1) Because this shader uses a raymarch the overall performance is sensitive to screen size and the number of iterations.
2) There is a deadspot directly perpendicular to the light plane's up direction.
3) 