---
layout: post
title:  "Neography Fonts Project"
summary: "Font design using code"
author: fetlach
date: '2025-03-17'
category: ['projects']
tags: projects
thumbnail: /assets/img/post_neography/VagaboundND.png
keywords: Conlang, Font, Neography, Scripting
usemathjax: false
permalink: /blog/neography-fonts-3-17-2025/
---
<h2> Background </h2>

During my undergrad I played through Tunic and became interested in neography.
Neography is a neat little field of design nestled in between humanities and art where the goal is to create "constructed languages" - for those unfamiliar, it's essentially a name given for making writing systems like Tolkien's elvish language.
I don't have a great understanding of the patterns behind encoding spoken language into written text, but I do have a knack for coding and my design skills aren't half-bad so exploring the topic made for a fun series of little projects nevertheless for sprinkling into other work.


<h2> First Drafts </h2>

To start I was interested in the idea of designing a conlang that was encoded english and strode the line of being just undecipherable enough to not crack under brute-force cryptographic analysis.

I designed these first fonts, "Bark" and "Alcheme W/B", manually along with a small transcoder commandline program to do the heavy lifting for encoding strings. Once they were finished I figured I wouldn't want to do something more complex due to the tedium it takes to ensure glyphs line up properly.

<center><img src="/assets/img/post_neography/Bark&Alcheme.PNG" class="img-fluid"></center><br>

Another idea I shelved due to the complexity, "Whine", was a blend between caligraphy and korean text - the design was based on combining basic "jamo" characters into compound blocks and running a spline path over the arrangement to simulate caligraphy. <br> While I was able to get a demo working that looked pleasing, I wasn't entirely happy with the result as it would be troublesome to use and render in most situations.

<center><iframe width="560" height="315" src="https://www.youtube.com/embed/ugdnu3Dz1Cs?rel=0&amp;controls=0&amp;showinfo=0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center>

For a while this was the end of the project. I liked the idea of a font with a combinatorial element to it, but was unable to do so directly as I had little experience at the time with font formats and font tools. I wasn't able to hack a solution together either as per-glyph transformation isn't typically supported as a feature in text renderers and manually creating dozens of glyphs with substitution rules is painstakingly slow in many font tools.


<h2> Procedural Generation </h2>

However, after looking into the tools further I found that some offer scripting APIs. <br>
These scripting APIs were definitely not designed for what I was looking to use them for, but they would suffice nevertheless.
So after writing out my font rules in python and downloading the fontforge libraries, I was ready to start taking my original designs to the next level. 

Now that I had a means of automating repetitive parts of designing a font with combinatorial characters, creating one seemed more feasible. 

To start, I revisited my basic "Bark" design and came up with nine 2-stroke glyphs that could then be combined into a set of 4-stroke glyphs by applying 2D transformations.

<center><img src="/assets/img/post_neography/Bark2Strokes.PNG" class="img-fluid"></center><br>

Combining the 2-stroke glyphs in fontforge was easy enough with a few workarounds. By copying the base glyphs into a temporary location, applying transformations, and copying the results into the final glyph, all the results were quickly done in a matter of seconds which would manually take a day or two. 
On the first pass some of the results required small manual touchups due to issues such as extraneous anchors, small gaps, or bad arrangements. While these can be mitigated somewhat by taking into account floating point innaccuracy and having a more robust script for combining the glyphs, it is to be expected with procedural generation - Though some of the outputs were unexpectedly better or revealed better designs because of the flaws.

<center><iframe width="560" height="315" src="https://www.youtube.com/embed/Joo10cFuVy0?rel=0&amp;controls=0&amp;showinfo=0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center>

Then came the issue of combining the base glyphs while typing. For this, we can make use of the GSUB table's ligature feature. A second script can propagate the lookup table easily, and afterwards the base glyphs combine automatically into their 4-stroke counterparts when typing!
While I didn't do it initally, using "combining" characters to designate ligatures is considered best practice as it allows for more complex ligature setups and allows you to use easily use both the plain and combined versions of the glyphs without needing workarounds. This isn't important in the context of this specific project but is good to note in case you want to do this yourself. 


<h2> Things to consider for web usage </h2>

There's also several issues I found while attempting to use this font on this portfolio site. 
Certain characters should be avoided like the plague due to contextual relevance. Spaces, graves, escape characters, and the like should not be touched as they run a high risk of breaking either your backend or the layout step in the text renderer. 
For instance; I used the backslash as one of my base glyphs and had to redo a good portion of my work as the backslash by default is treated as both an escape character and component of special characters in most programming languages. This can cause a lot of side effects, as I learned when my webpage in testing prompted me out of the blue to print itself.

```javascript
// An example of a "problem" string:
inString = r"-/\\--\\-///-////\\///---\\-\\\\-\\/--///--/--\\///--/\\----\\\\/\\//\\\\\\/-\\-/-\\--\\/--///-/--\\/-\\\\-\\\\--/-\\/\\-/\\\\////----/\\--/\\/-//--\\"
// This one doesn't break the browser though since I've doubled each backslash
```

Sticking to alphanumerics and extra encoding slots for glyphs saves a lot of headache in the long run.

Additionally, since neography isn't generally an expected use case you'll probably run into some other strange issues nevertheless. <br>
In my case, it was cross-platform compatability which gave me trouble. For one reason or another, IOS safari browsers in particular doesn't like custom GPOS and GSUB tables. While the font was correctly positioned and rendered with kerning and ligatures, it was not sized correctly in the layout step. <br>The two versions of the test page shown below were from Chrome on windows 64 and IOS respectively, yet only on the IOS version does the issue occur despite both using webkit for text rendering.

<center><img src="/assets/img/post_neography/IOSvsNotFontComparison.PNG" class="img-fluid"></center><br>

I sidestepped around this issue for the meantime by writing a script to convert my input string into a version with the explicit characters for the combined glyphs, though may properly address it in the future by setting up a debugging environment. Always make sure to test!
