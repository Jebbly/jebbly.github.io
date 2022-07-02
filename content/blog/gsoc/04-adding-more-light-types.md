---
title: Adding More Light Types
date: "2022-06-25"
description: "How different light types in Blender are used in the many lights sampling algorithm."
---

In this post, I'll be discussing how the different light types in Blender were incorporated into the many lights sampling algorithm. Some of these lights could be immediately plugged in to the existing work, but others needed some redesigning of the logic.

## Different Light Types

Blender has a variety of light types that all need to be supported by the algorithm. This includes:
- Point, Spot, and Area Lights
- Emissive Triangles
- Distant and Background Lights 
Each of these bullet points have slightly different places in the code, which I'll elaborate on below.


### Point, Spot, and Area Lights

Once we have point lights working, it's really immediate to also incorporate spot and area lights. All we have to do is update our light tree construction to account for their different bounding information (actually, I had to update some of the traversal code as well, but the next section will explain why I don't discuss it here). The first thing to get out of the way is that the energy calculations for these light types are exactly the same. Next, this is the different bounding cone information for each light type:

| Light Type      | Axis | $\theta_o$ | $\theta_e$ |
| :----: | :----: | :----: | :----: |
| Point | Arbitrary | $\pi$ | $\pi / 2$ |
| Spot | Spotlight Direction | 0 | Spotlight Angle |
| Area | Normal Axis | 0 | $\pi / 2$ |

This makes sense because point lights don't have a defined orientation direction while spot lights and area lights do. Implementing this in the code is also very straightforward using a bunch of conditionals. 

The bounding box information for these lights is only slightly more interesting, and there are only a few things to note. The point lights and spot lights have an associated ``size``. What this means is that they can actually emit light from a radius of that size, so our bounding box needs to account for that radius. On the other hand, area lights can either be a disk or a rectangle, and thus have 2 dimensions to them. They have the members ``axisu`` and ``axisv`` which correspond to the orientation of those dimensions, as well as a ``sizeu`` and ``sizev`` which dictate how far along the axis it goes. Knowing this, the code is also relatively straightforward:
```cpp
// intern\cycles\scene\light_tree.cpp
Light *lamp = scene->lights[lamp_id];
LightType type = lamp->get_light_type();
const float3 center = lamp->get_co();
const float size = lamp->get_size();

if (type == LIGHT_POINT || type == LIGHT_SPOT) {
  /* Point and spot lights can emit light from any point within its radius. */
  const float3 radius = make_float3(size);
  bbox.grow(center - radius);
  bbox.grow(center + radius);
}
else if (type == LIGHT_AREA) {
  /* For an area light, sizeu and sizev determine the 2 dimensions of the area light,
  * while axisu and axisv determine the orientation of the 2 dimensions. 
  * We want to add all 4 corners to our bounding box. */
  const float3 half_extentu = 0.5 * lamp->get_sizeu() * lamp->get_axisu() * size;
  const float3 half_extentv = 0.5 * lamp->get_sizev() * lamp->get_axisv() * size;

  bbox.grow(center + half_extentu + half_extentv);
  bbox.grow(center + half_extentu - half_extentv);
  bbox.grow(center - half_extentu + half_extentv);
  bbox.grow(center - half_extentu - half_extentv);
}
```

In other parts of the Cycles code, the area light's size is also scaled by the ``size`` member. I've always seen this factor equal to 1.0 in my own debugging, but I've left it here just to be safe.


### Emissive Triangles

Even though I have a separate section for emissive triangles, I'm not really going to talk about the bounding information calculation (most of it was based off the past GSoC work anyways). Instead, this is going to be about how emissive triangles made me realize that some of the traversal logic needed to be reconsidered.

Originally, I didn't think there would be anything too special about triangle lights besides using the ``prim_id`` to distinguish them during light tree construction. However, when I got to traversal, I encountered a slight issue: although I could differentiate between a normal light source and an emissive triangle using the ``light_distribution`` array, I still didn't have enough information to calculate the importance. It's still possible to get the triangle's vertices to manually calculate the bounding box min/max and also take the cross product to find the orientation axis. But then there's also the issue of finding a proper energy estimate.

In any case, doing all of this work during traversal seems like a huge performance issue. Additionally, this information is stuff that we can calculate at construction time and then store for the future. So using the same idea as the ``device_vector<KernelLightTreeNode> light_tree_nodes``, there's an array on the device containing the bounding information for each emitter:

```cpp
// intern\cycles\kernel\types.h
typedef struct KernelLightTreeEmitter {
  /* Bounding box. */
  float bounding_box_min[3];
  float bounding_box_max[3];

  /* Bounding cone. */
  float bounding_cone_axis[3];
  float theta_o;
  float theta_e;

  /* Energy. */
  float energy;

  /* prim_id denotes the location in the lights or triangles array. */
  int prim_id;
  union {
    struct {
      int shader_flag;
      int object_id;
    } mesh_light;
    struct {
      float pad;
      float size;
    } lamp;
  };

  /* Padding. */
  int pad1;
} KernelLightTreeEmitter;
static_assert_align(KernelLightTreeEmitter, 16);
```

The information under ``prim_id`` is the same as the information from the light distribution. However, by keeping it inside of this struct, we can remove our light tree kernel's dependency on the light distribution. There still is a lot of overlap between this struct and the ``KernelLightTreeNode`` struct, but it works for now. Now after our construction has sorted all of the primitives in order, we can fill out the corresponding bounding information:

```cpp
// intern\cycles\scene\light.cpp
KernelLightTreeEmitter *light_tree_emitters = dscene->light_tree_emitters.alloc(num_distribution);
for (int index = 0; index < num_distribution; index++) {
  LightTreePrimitive &prim = light_prims[index];
  BoundBox bbox = prim.calculate_bbox(scene);
  OrientationBounds bcone = prim.calculate_bcone(scene);
  float energy = prim.calculate_energy(scene);

  light_tree_emitters[index].energy = energy;
  for (int i = 0; i < 3; i++) {
    light_tree_emitters[index].bounding_box_min[i] = bbox.min[i];
    light_tree_emitters[index].bounding_box_max[i] = bbox.max[i];
    light_tree_emitters[index].bounding_cone_axis[i] = bcone.axis[i];
  }
  light_tree_emitters[index].theta_o = bcone.theta_o;
  light_tree_emitters[index].theta_e = bcone.theta_e;

  if (prim.prim_id >= 0) {
    light_tree_emitters[index].mesh_light.object_id = prim.object_id;

    int shader_flag = 0;
    // query shader flags (same as light distirbution)
    light_tree_emitters[index].mesh_light.shader_flag = shader_flag;
  }
  else {
    Light *lamp = scene->lights[prim.lamp_id];
    light_tree_emitters[index].lamp.size = lamp->size;
    light_tree_emitters[index].lamp.pad = 1.0f;
  }
}
dscene->light_tree_emitters.copy_to_device();
```

The advantage of this approach is that all of the decision making is happening at construction time. During traversal, we don't need any conditionals to handle a different construction for each type of light. We just trust that all the information has been calculated correctly beforehand and then directly plug it into our formula. 

The last thing to do is to adjust the triangle sampling PDF. If you recall in my last post, I mentioned that the light distribution will pre-calculate some of the PDF. For example, for light sources, it knows that it'll be sampling uniformly over light samples, so it sets 
```cpp
// intern\cycles\scene\light.cpp
kintegrator->pdf_lights = 1.0f / num_lights;
``` 
On the other hand, the light distribution samples triangles relative to their total area. This is something that varies per-triangle, so the best that can be done is to pre-compute ``kintegrator->pdf_triangles`` as ``1.0f / trianglearea``. Then the contributing PDf is calculated during ``triangle_light_sample()``:
```cpp
// intern\cycles\kernel\light\light.h
const float pdf = area * kernel_data.integrator.pdf_triangles;
```
We'll also be using ``triangle_light_sample()``, but that's not going to be the PDF of our sampling method. Instead, we set ``kintegrator->pdf_triangles`` and then divide ``ls->pdf`` by the triangle's area to counteract the multiplication done inside of the function. This essentially converts the pre-computed PDF to ``1.0f``, so now we're free to control the PDF appropriately.


### Distant and Background Lights

The reason why distant lights and background lights need to be handled separately is because the light tree is inherently location-based. Since these lights can be considered infinitely far away, we can't really construct a bounding box or anything to make them part of the light tree. The original method we wanted to implement was to first pick a light from a light tree and another light from the distant/background lights, and then choose one of the two after weighing their importances. The idea would be that having 2 specific lights would be more specific.

However, halfway through implementing this, I discovered that this would actually be pretty complicated. This is because we not only need to calculate the probability of selecting the light in order to scale our PDF accordingly. Now suppose we select one object from $A = \{A_1, A_2\}$ and one object from $B = \{B_1, B_2\}$. Then we put our two selected objects into a new group $C$ and select one out of the two. For the sake of shorter notation, let $O_{i_N}$ denote the probability of selecting object $O_i$ from group $N$. Now the probability of ending up with $A_1$ as our final selection would be:
$$
\mathbb{P}(A_{1_C}) = \mathbb{P}(A_{1_C} | A_{1_A} \cap B_{1_B}) \cdot \mathbb{P}(A_{1_A} \cap B_{1_B}) + \mathbb{P}(A_{1_C} | A_{1_A} \cap B_{2_B}) \cdot \mathbb{P}(A_{1_A} \cap B_{2_B})
$$
Technically there are a few more terms (cases where $A_2$ is selected from group $A$) but we can ignore them because they're all equal to $0$. The general idea is that to find the actual probability, we'd have to partition the probabilities into cases. So in this case, the true probability of selecting $A_1$ is the probability of selecting it when $C = \{A_1, B_1\}$ plus the probability of selecting it when $C = \{A_1, B_2\}$. In code, we'd be able to naturally find the value of a single one of these terms, but we'd have to do a lot of extra computation to find the others.

The next best thing we can do is first decide whether we want to sample from the light tree or to sample from the distant lights. For now, the easiest way to do this is by examining their relative energies. The advantage to this approach is that we can pre-compute both of these during construction time, but in the future, we may want to introduce an appropriate importance heuristic to decide between the two. Here, ``pdf_light_tree`` is calculated as the relative energy of the light tree compared to the total energy involved:
```cpp
// intern\cycles\kernel\light\light_tree.h
float tree_u = path_state_rng_1D(kg, rng_state, 1);
if (tree_u < kernel_data.integrator.pdf_light_tree) {
  pdf_factor *= kernel_data.integrator.pdf_light_tree;
  ret = light_tree_sample<false>(
      kg, rng_state, randu, randv, time, N, P, bounce, path_flag, ls, &pdf_factor);
}
else {
  pdf_factor *= (1 - kernel_data.integrator.pdf_light_tree);
  ret = light_tree_sample_distant_lights<false>(
      kg, rng_state, randu, randv, time, N, P, bounce, path_flag, ls, &pdf_factor);
}
```

The downside to this approach is that we'll have to perform a linear scan if we want to sample from the distant lights group. Realistically though, or at least from my perpective, most scenes shouldn't have that many distant lights. Furthermore, we can also compute importance heuristics if we choose to sample from the distant light group, so we can make more informed decisions about which light to sample. 

For now, ``light_tree_distant_light_importance()`` only returns the energy of the given distant light:
```cpp
// intern\cycles\kernel\light\light_tree.h
const int num_distant_lights = kernel_data.integrator.num_distant_lights;
float total_importance = 0.0f;
for (int i = 0; i < num_distant_lights; i++) {
  total_importance += light_tree_distant_light_importance(kg, P, N, i);
}
const float inv_total_importance = 1 / total_importance;

float light_cdf = 0.0f;
float distant_u = path_state_rng_1D(kg, rng_state, 1);
for (int i = 0; i < num_distant_lights; i++) {
  const float light_pdf = light_tree_distant_light_importance(kg, P, N, i) *
                          inv_total_importance;
  light_cdf += light_pdf;
  if (distant_u < light_cdf) {
    *pdf_factor *= light_pdf;
    ccl_global const KernelLightTreeDistantEmitter *kdistant = &kernel_data_fetch(
        light_tree_distant_group, i);

    const int lamp = -kdistant->prim_id - 1;

    if (UNLIKELY(light_select_reached_max_bounces(kg, lamp, bounce))) {
      return false;
    }

    return light_sample<in_volume_segment>(kg, lamp, randu, randv, P, path_flag, ls);
  }
}
```
This is bound to change as we come up with better heuristics in the future.


## Closing Thoughts

Thanks to the heavy debugging from the work with point lights, most of the math was pretty much working from the get-go. However, there's still a lot of optimizations to the heuristics that can (and will) be made. My main concern at the moment is that these heuristics don't take visibility into consideration, which can really hurt the sampling in extreme cases. For example in one case, we could be placing high importance on one group of lights and dedicating a lot of samples towards them, without realizing that they're actually all occluded! We'll have to have another discussion for this in the future, but one solution that comes to mind is to also randomly select between using the light tree sampling and using the default light distribution sampling. 

Secondly, I also realized that there are 3 additional functions to update, which are used when Cycles performs indirect light samples (I'll be making a separate post about this). These functions are basically used when Cycles is sampling based off of the BSDF and the sample intersects a light source, so we need to calculate what the direct lighting's PDF would be in order to weight the multiple importance sampling. The functions are:
- ``background_light_pdf()``
- ``triangle_light_pdf()``
- ``light_sample_from_intersection()``
These functions are pretty self-explanatory, but it'll be a little tricky to incorporate the light tree into them. More on that in the next post!