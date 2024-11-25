---
layout: post
title: Reflecting on GSoC 2022
date: 2022-12-19
description: A three month late reflection on GSoC 2022.
tags: path-tracing
categories: gsoc-2022
images:
  compare: true
  slider: true
---

I was planning on writing this post a lot sooner, but work for the fall semester hit me really fast. Anyways, now that GSoC is over (although, it's already been over for more than 3 months), you can read my final report [here](https://wiki.blender.org/wiki/User:JeffreyLiu/GSoC2022/FinalReport). Seeing the eye candy at the end was really fun! Here's my favorite result from GSoC, which is also shown in the report:

Here's the render after 30 seconds of rendering using the master branch (left), compared to 30 seconds of rendering on the GSoC branch (right):

<img-comparison-slider>
  {% include figure.liquid path="assets/img/blog/07-attic-original.png" class="img-fluid rounded z-depth-1" slot="first" description="Cycles Implementation" %}
  {% include figure.liquid path="assets/img/blog/07-attic-mls.png" class="img-fluid rounded z-depth-1" slot="second" description="New Implementation" %}
</img-comparison-slider>

There's still some noise, but it's a lot better. Since I'm also writing this extremely late, I get to say that other people have picked up the work and got it merged into master!


## Closing Thoughts

I've probably said it many times already, so I'll try to keep it short. All I can say is that Google Summer of Code was everything I wanted and expected it to be and more. Blender basically defined me as a teenager so being able to contribute to it has felt incredible. I'm really thankful for everyone who took the time to read my posts, upload test scenes, provide advice/resources, and contribute code. The whole process has taught me a lot and helped me develop more confidence as a computer scientist/programmer. Every company I've interviewed at this semester has also asked me about working with Blender, so I think it's safe to say that it's also helped me with my job search.

Moving on, although the main parts of the code have already been merged into master, there are still plenty of bugs and improvements to be worked on. I'll be working on those from now on, and after those are done, hopefully I can also help out with another Blender/Cycles project!
