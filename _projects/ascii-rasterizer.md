---
layout: page
title: ASCII Rasterizer
description: Interactive (CPU) software rasterizer with CLI display.
img: assets/img/projects/ascii-rasterizer/suzanne-thumbnail.png
importance: 2
category: graphics
related_publications: false
---

This software rasterizer was completed with a few other teammates for UIUC's CS 128 final project, which was an excuse for me to explore something I'd wanted to build for a while at this point. I wanted to do something a bit more than a normal rasterizer, so I was inspired by [donut.c](https://www.a1k0n.net/2011/07/20/donut-math.html) to use the command-line as the choice of display.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/ascii-rasterizer/suzanne-thumbnail.png" title="Sample Screenshot" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The executable can load simple OBJ files, like Blender's Suzanne! This project surprised me with how... interesting Suzanne's topology is.
</div>

At its core though, it's still just a rasterizer that discretizes color values to the scale `` .:-=+*#%@``.

The code is also available on [GitHub](https://github.com/Jebbly/Software-Rasterizer/) to play around with.
