---
layout: page
title: Hand Tracker
description: Stereo hand tracker with multi-CNN prediction and triplane fusion.
img: assets/img/projects/hand-tracking/heatmap.png
importance: 1
category: vision
related_publications: false
---

This project was done with another classmate as part of Professor Shenlong Wang's UIUC CS 498, Machine Perception course. Tracking hands in real-time is a surprisingly difficult problem, especially if you don't want to add additional hardware (e.g., active depth sensors, etc.). Our proposed solution was a combination of two papers:

1. "[3D Hand Pose Tracking and Estimation Using Stereo Matching](https://arxiv.org/abs/1610.07214)" for hand segmentation and depth estimation,
2. "[Robust 3D Hand Pose Estimation in Single Depth Images: from Single-View CNN to Multi-View CNNs](https://arxiv.org/abs/1606.07253)" for joint estimation.

I helped a bit with (1) for hand segmentation, but my main contribution was re-implementing (2). The general idea of this is to use the estimated hand depths, reproject them into 3D space, use [PCA](https://en.wikipedia.org/wiki/Principal_component_analysis) to extract a bounding box, and then project them onto the three bounding box planes.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/hand-tracking/projections.png" title="Projected Point Clouds" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    On the left, the depth image. On the right, the reprojected point clouds on the three planes (with hand joints highlighted on the bottom).
</div>

After extracting the triplane projections, we then run a multi-CNN on each projection to extract heatmaps of all 21 hand joints.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/hand-tracking/multi-cnn.png" title="Multi-CNN Pipeline" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    A diagram from the second paper outlining the multi-CNN architecture. Each projection produced an 18x18x21 output (18x18 heatmaps for 21 joints).
</div>

The heatmaps, which indicate the confidence of where a joint might be, are fused back with each planar projection to output a final prediction. This sounds great in theory because we get to manually extract a bit more information from just raw depth images, but in practice, we ran into a lot of issues similar to mode collapse.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/hand-tracking/mode-collapse.png" title="Mode Collapsed Output" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The multi-CNN finds a local minimum and always outputs basically the same thing, even for different joints and different projections.
</div>

Unfortunately, we ran out of time to fully fix this issue and put everything in an end-to-end pipeline (the PCA and projection was done in C++, the multi-CNN was done in PyTorch, and the triplane fusion was done in C++ again). However, we did find that the average error on our test set was only around 51 millimeters, which might be acceptable depending on the use case.

All of our code can be found on [GitHub](https://github.com/hungdche/Stereo-Hand-Tracking). Although I don't plan on revisiting this specific project in the near future, I still think it's still interesting to explore how triplanes are used for 3D understanding (they seem to have some use in the area of 3D content generation as well, through [GANs](https://nvlabs.github.io/eg3d/) and [diffusion models](https://jryanshue.com/nfd/)).
