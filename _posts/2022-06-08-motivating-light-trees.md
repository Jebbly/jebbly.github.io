---
layout: post
title: Motivating Many Lights Sampling
date: 2022-06-08
description: An introduction to the many lights sampling algorithm and how I plan to implement it in Blender.
tags: path-tracing
categories: gsoc-2022
---

To start off, I'll try to describe the motivation for the [many lights sampling algorithm](http://www.aconty.com/pdf/many-lights-hpg2018.pdf). In general for Monte Carlo estimation, we want to find ways to obtain the same result with fewer samples and/or obtain more precise results with the same number of samples.

## Importance Sampling

One way of doing this is through importance sampling, which is when we try to use sampling PDFs that scale according to the function itself. PBRT does a pretty good job at explaining the intuition behind this technique if you don't believe me.

In the context of light sampling, we're estimating the integral in the rendering equation:

$$
L_o(p, \omega_o) = L_e(p, \omega_o) + \int_\Omega f(x, \omega_i, \omega_o) L_i(p, \omega_i) \,d \omega_i
$$

If we only consider direct lighting for now, i.e. $$L_i(P, \omega_i)$$ is from a light source, then importance sampling can be pretty important. Let's say we're at an intersection point and we want to find the contribution of all the direct lights. Mathematically speaking, we could just uniformly sample the hemisphere of solid angles and check if there's a light in that direction. The issue with this approach is that most of the samples are probably going to hit nothing, and even worse, there's technically a 0% chance of sampling a light with no area (e.g. a point light).

Alternatively, one basic approach is to instead sample from the light sources first. This way, if we exclude visibility for now, we can guarantee that our samples are going to be non-zero. This also means we can include the contributions from lights with no area.


## The Many Lights Sampling Algorithm

So sampling using light source information is already a step in the right direction, but there's still room for improvement. For example, shouldn't we place less emphasis on lights that are either really weak, far away, or facing the other direction? That's where the many lights sampling algorithm comes into play.

The many lights sampling algorithm first constructs a sort of BVH that contains information about the light sources. Starting from the top-level node, it uses some heuristics to collect neighboring lights into two groups, and provides a general idea of the groups' total energy and orientation. The paper describes a measure to determine the cost of grouping certain lights, so we try to pick the split that minimizes this cost before converting the two groups into child nodes.

Once the tree is constructed, we can traverse it when we're sampling light sources. At each level of the light BVH, we use our heuristics to determine which child we want to examine next. These heuristics will guide us in the general direction towards lights that are going to contribute more towards our final result. As a result, we can introduce a substantially larger number of lights into the scene, which would have previously produced poor results with the basic importance sampling technique.

One last thing that the algorithm does is to force splitting during traversal. This is because sometimes, especially in the upper levels of the BVH, a general idea of the total energy and orientation isn't enough information to decide which child is traverse towards. As a result, some splitting threshold is used so that if the normalized variance of the node is greater than the threshold, we just traverse both children just to be safe.


## Implementation Plan

As discussed with Brecht and Lukas, the goal of this project is to progressively add support for more things. In other words, I'll be starting with just point lights first, and once those are working, I'll move on to other types. The first step is to actually construct the light BVH on the host, using information about the lights in the scene. Although this did take a while to implement, there's nothing too interesting. It's almost all contained in ``cycles/scene/light_tree.h`` and ``cycles/scene/light_tree.cpp``, where there's a few extra structs to help out with construction. Otherwise, the overall logic follows the construction of a BVH in PBRT, but the SAH is replaced with the surface area orientation heuristic (SAOH) described in the paper.

Once the construction is completed on the host side, there has to be a way to transfer this information to the device for traversal. The one caveat is that the device must take in a linear array of data, so we have to flatten the light BVH nodes. To traverse the interior nodes as a flat array, we need the following information:
- Index of the second child
- Bounding box of the light cluster
- Bounding cone of the cluster as described in the paper (orientation axis, $$\theta_o$$, $$\theta_e$$)
- Total energy and energy variance of the cluster

Note that here, we can construct the array such that the first child is always adjacent to the parent node. As a result, we only need to store the index of the second child. We also keep the leaf nodes as a separate flat array because they store slightly different information. Once we're at the leaf nodes, we need to know the following:
- Number of lights in the leaf
- Index of the first light in the light array
- Bounding box of the cluster
- Bounding cone of the cluster
- Total energy of the cluster

We can use the index of the second child to go from the interior node array to the leaf node array. if it's positive, we can assume the children are still interior nodes. Otherwise, if it's negative, then we can take the negative and index into the leaf onde array. Although there is some overlap between the interior and leaf nodes, we should be able to save some memory by avoiding storage of unused information. Once we're able to figure out how to traverse on the device, the last step should just be to figure out how to adjust the light PDF so that our estimators remain unbiased.


## Closing Thoughts

Even though it seems relatively short now, this post took a really long time to write. At first, I wanted to go into more detail about multiple importance sampling and then prove some of PBRT's methods for sampling direct lights. However, after two hours, I realized that I was still regurgitating a lot of information that can already be found on PBRT or Veach's thesis. As a result, I decided to re-pivot so I could talk about a subject that's hopefully a little more specific and focused.

I also wasn't really sure how long the post should be, so I tried to describe more of the intuition behind the algorithm and just a high-level overview of the implementation plan. There are some technical details about the implementation that I was hoping to cover, so I'll try to make a dedicated post about the Blender side of things next time. Otherwise, if anyone has taken the time to read this far, please do let me know any errors I made, how I can improve, or what you'd like to see next!
