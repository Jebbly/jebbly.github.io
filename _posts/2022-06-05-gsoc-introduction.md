---
layout: post
title: Introduction to GSoC with Blender
date: 2022-06-05
description: A self-introduction and my plans for this series.
tags: path-tracing
categories: gsoc-2022
---

Welcome to the first post of my blog! This is something I've always kind of wanted to do but wasn't sure where to start or what to post. Fortunately, participating in Google's Summer of Code (GSoC) has finally given me the push that I needed to get going.

This summer, I'll be working with the Blender Foundation. I've been a long-time Blender user and it's something that I've wanted to give back to for a long time (in fact, it's part of the reason why I started learning C++). Part of GSoC with Blender also requires me to post weekly reports of my progress. I'll be doing those summaries on the Blender wiki, but I wanted to keep a complementary blog, which will be on the more technical side, for 2 main reasons.

Firstly, I want somewhere to keep track of my notes and my progress. Secondly, since this series is definitely going to be a sort of learning journey, I'm hoping that readers will be able to refer to it as a resource to get familiar with the Cycles codebase. It would be great if it helped new contributors get started.


## The Project

The project I'm working on is to integrate Many Lights Sampling into Cycles X, which is the most recent version of Blender's production renderer. I'll be mentored by both Brecht Van Lommel and Lukas Stockner. If this project sounds familiar, that's because past GSoC participants have already done work with this algorithm. This year, the goal is to complete the implementation and hopefully get it merged.

Currently, I've been reading through PBRT's chapters about multiple importance sampling (MIS) and direct light sampling. I've also checked out some of Blender's source code related to sampling lights. The first step of implementing this algorithm is to get it working for point lights only, so I've created some Blender scenes with a single cube combined with 1, 2, 4, 8, 100, or 1000 point lights (no background light yet).


## Closing Thoughts

Since this is my first time writing a blog, it definitely felt strange; I spent an unnecessarily long amount of time deciding how I wanted to express certain things. Hopefully, it'll start to feel more natural once I write more posts.

In my next post, I'm planning to elaborate on what I've learned so far about MIS and Blender's source code. Hopefully I'll also have the algorithm working with point lights so I can talk about that too!
