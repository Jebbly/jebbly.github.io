---
layout: page
title: Style Share
description: Web-based 3D engine with neural stylization post processing.
img: assets/img/projects/style-share/sample-thumbnail.png
importance: 1
category: graphics
related_publications: false
---

This was created as a team project for UIUC's CS 222. We tried to combine a mish-mash of ideas together and this was the result. I've always been interested in stylized rendering, so we tried to create a 3D editor to arrange a basic scene and then use a model to transfer style content on to it. I was mainly responsible for the 3D editor, which was done in [React-three-fiber](https://r3f.docs.pmnd.rs/getting-started/introduction) (I claim little experience with React and/or web-dev... this was just a bunch of hacking documentation around).

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/style-share/editor.png" title="Editor" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The 3D editor has some basic functionality, including loading an environment map, uploading OBJ files (with no material...), arranging objects around, and visualizing a grid. The idea was to keep it pretty simple and rely on style transfer to finish the job.
</div>

The current viewport could be saved as either the content image _or_ the style image, although content makes much more sense. We then used [an optimized TensorFlow style transfer model](https://blog.tensorflow.org/2020/04/optimizing-style-transfer-to-run-on-mobile-with-tflite.html) as a post-processing step on the content image.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/style-share/stylized-image.png" title="Stylized Image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Results are... mixed. The model generally seems to "wash" the entire image with a certain color that's usually quite pale.
</div>

If you don't want to manually create a scene, you also have the option of uploading a pre-created content image. There's also a [live demo](https://style-transfer-backend-ix3zc64heq-uc.a.run.app/) to try it out, with a gallery of previously stylized photos.

It generally seems pretty hard to reproduce the quality of the results from the TensorFlow post, but it might also be the set of content images I've tested with. I'd love to play around with this more in the future though :)
