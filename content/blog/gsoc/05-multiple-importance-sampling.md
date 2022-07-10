---
title: Correcting for Multiple Importance Sampling
date: "2022-07-09"
description: "Fixes to the PDF calculations to adjust for the light tree sampling."
---

If you recall from my second post, importance sampling is the name of the game, but I actually kind of omitted something from that chapter. Notice that there's actually two main terms in the integrand of the rendering equation:
$$
L_o(p, \omega_o) = L_e(p, \omega_o) + \int_\Omega f(x, \omega_i, \omega_o) L_i(p, \omega_i) \,d \omega_i
$$
So it's great if we're looking for light sources that are really strong and contribute a lot to the $L_i$ term, but there's a slight issue. For example, consider a scenario where we're looking across a mirror from a very shallow angle and there's a really strong light source that's directly above the mirror. Turns out, that light source isn't really contributing much to what we're seeing at all! The issue is that the BSDF term also plays a role in our integrand, and we're going to have a lot of trouble if we keep focusing our samples towards a strong light source and isn't contributing all that much. Thankfully, people have come up with a pretty good solution: multiple importance sampling!


## Multiple Importance Sampling

The idea of multiple importance sampling is to take some samples based on both terms of the integrand (in this case, $f$ and $L_i$). As PBRT mentions, we can't just define two separate estimators that each use a different sampling distribution because the variance is additive. Instead, we use the estimator given by:
$$
\frac{1}{n_f} \sum_{i = 1}^{n_f} \frac{f(X_i) g(X_i) w_f(X_i)}{p_f(X_i)} + \frac{1}{n_g} \sum_{i = 1}^{n_g} \frac{f(X_i) g(X_i) w_g(X_i)}{p_g(X_i)}
$$
Here, $n_f, n_g$ are the number of samples used from the $p_f, p_g$ distributions respectively. The $w_f$ and $w_g$ terms correspond to a weighting heuristic (that should sum to 1). It's relative straightforward to verify this estimator is still unbiased. I won't :
$$
\begin{aligned}
\mathbb{E}[F] 
&= \mathbb{E}\left[\frac{1}{n_f} \sum_{i = 1}^{n_f} \frac{f(X_i) g(X_i) w_f(X_i)}{p_f(X_i)} + \frac{1}{n_g} \sum_{i = 1}^{n_g} \frac{f(X_i) g(X_i) w_g(X_i)}{p_g(X_i)} \right] \\
&= \frac{1}{n_f} (n_f) \mathbb{E}\left[ \frac{f(X_i) g(X_i) w_f(X_i)}{p_f(X_i)} \right] + \frac{1}{n_g} (n_g) \mathbb{E}\left[ \frac{f(X_i) g(X_i) w_g(X_i)}{p_g(X_i)} \right] \\
&= \int \frac{f(x) g(x) w_f(x)}{p_f(x)} p_f((x) \,dx + \int \frac{f(x) g(x) w_g(x)}{p_g(x)} p_g(x) \,dx \\
&= \int f(x) g(x) w_f(x) \,dx + \int f(x) g(x) w_g(x) \,dx \\
&= \int f(x) g(x) (w_f(x) + w_g(x)) \,dx \\
& = \int f(x) g(x) \,dx.
\end{aligned}
$$

In Cycles, each pass of ``integrator_shade_surface()`` actually takes two samples: one for direct lighting (only sampling light sources) and one for indirect lighting (sampling based on the BSDF). The weighting used is the power heuristic, which simplifies slightly because $n_f = n_g = n$ in this case:
$$
w_f(x_i) = \frac{(n_f p_f(x_i))^2}{(n_f p_f(x_i))^2 + (n_g p_g(x_i))^2} = \frac{n^2 p_f(x_i)^2}{n^2(p_f(x_i)^2 + p_g(x_i)^2)} = \frac{p_f(x_i)^2}{p_f(x_i)^2 + p_g(x_i)^2}.
$$
The same calculation can be done for $w_g(x_i)$. The important thing to note here is that, in order to calculate the proper power heuristic for a given sample, we actually need to compute the direct light PDF as well as the indirect light PDF. We're not going to worry too much about the indirect light PDF because we're not changing anything there, so the rest of this post is focused on how the direct light PDF needs to be adjusted.


## Many Lights Sampling PDFs

In the default implementation where direct lighting is sampled based on the light distribution (as I've mentioned many times before), it's pretty simple to compute the PDF. In fact, most of it is already pre-computed during construction:
- Light sources have a ``1 / num_lights`` probability of being chosen
- Triangles have an ``area / total_area`` (where ``total_area`` is the total area of all the triangles) probability of being chosen
- If there are both light sources and triangles involved, divide those probabilities in half (because either a light or a triangle is chosen)
When we sample from the BSDF and we happen to intersect with a light source, we basically already have all this information. The only additional computation to be made is if we need to calculate a specific triangle's area.

Things get a little complicated, to say the least, when it comes to sampling from the light tree. The main issue is that the direct light sampling distribution is position-dependent. As a result, there isn't much information to compute during construction, and we need to calculate the probability of sampling that light source from whatever point we were sampling from. To review how we sample light sources:
- First, we sample either from the light tree or the distant light group (probability based on relative energies)
- If we choose the light tree, we traverse down the binary tree (according to their relative importances) until we hit a leaf node
  - At a leaf node, we sample based off of a CDF constructed from the relative importances of the light sources in the leaf
- If we choose the distant light group, we immediately sample based off of a CDF constructed from the relative importances


### A Brief Interlude
Before we even get to all those PDF calculations, the more pressing question is how to actually find the light source from the distant group or the light tree. It's easy to start from the top and pick a light source in the light tree, but what about the converse? Given some light source, how do we know where it would be in the light tree? The same issue holds for the distant lights group.

My first idea was to attach another member variable to the ``KernelLight`` struct, which would point back to the corresponding index in the light tree emitters array or the distant lights array. Something similar would need to be done for triangles. The issue with this approach is that the ``KernelLight`` struct is already perfectly aligned to 16 bytes, so adding another member variable (4 bytes for an int) would also require an additional 12 bytes of padding for alignment. Furthermore, not every triangle is an emitter, so it would be holding empty information. Lastly, this member is also useless if the light tree isn't being used. 

My next idea was to instead creating separate arrays which would store their corresponding indices in the light tree setup. For example, if we're at a triangle with index 0, then we query the triangle array at index 0 for its index in ``light_tree_emitters``. We do this for both light sources and triangles, but the corresponding indices for distant lights will index into the distant lights array:
```cpp
KERNEL_DATA_ARRAY(uint, light_to_tree)
KERNEL_DATA_ARRAY(uint, triangle_to_tree)
```
You may have noticed that this array needs to match the size of all the triangles, emissive or not, in order to work. The tradeoff here is that we're using up a bit of unnecessary memory, but the lookup times are fast and constant. Now that we know where we are in the light tree, we can start worrying about how to calculate the probability of reaching that location.


### Light Tree Group
The harder case is the light tree because the direct light sampling method relies on traversing down a binary tree before arriving at some light source; in the case of sampling a light from the indirect light sampling, we're given a specific light source and asked to calculate the probability of sampling it. To do this, we'll need a way to go back up the tree in order to compare importances at each level. The easiest way to do that is to update our `LightTreeNode` and `LightTreeEmitter` structs to store its parent index. It works out perfectly because our structs already had some extra padding anyways:
```cpp
// intern\cycles\kernel\types.h
typedef struct KernelLightTreeNode {
  ...

  /* Parent. */
  int parent_index;

  /* Padding. */
  int pad1;
} KernelLightTreeNode;
static_assert_align(KernelLightTreeNode, 16);

typedef struct KernelLightTreeEmitter {
  ...

  /* Parent. */
  int parent_index;
} KernelLightTreeEmitter;
static_assert_align(KernelLightTreeEmitter, 16);
```
This means we also need to update our light tree construction too. When we're finishing up construction during ``LightTree::flatten_tree``, we attach an extra argument to indicate the index of the parent:
```cpp
// intern\cycles\scene\light_tree.cpp
int LightTree::flatten_tree(const LightTreeBuildNode *node, int &offset, int parent)
{
  ...
  int current_index = offset;
  offset++;

  /* If current node contains lights, then it is a leaf node.
   * Otherwise, create interior node and children recursively. */
  if (node->num_lights > 0) {
    current_node->first_prim_index = node->first_prim_index;
    current_node->num_lights = node->num_lights;
    current_node->is_leaf_node = true;
  }
  else {
    current_node->num_lights = 0;
    current_node->is_leaf_node = false;

    /* The first child is located directly to the right of the parent. */
    flatten_tree(node->children[0], offset, current_index);
    current_node->second_child_index = flatten_tree(node->children[1], offset, current_index);
  }

  return current_index;
}
```
Note that the first call to this function passes an argument of ``-1`` to the ``parent`` parameter, so we know when we're at the root. Now that we have a way to traverse up the tree, we're ready to try calculating some probabilities! The first step is to find the probability of actually selecting the light when we're at the leaf node:
```cpp
ccl_device float light_tree_pdf(KernelGlobals kg, const float3 P, const float3 N, const int prim)
{
  float distant_light_importance = light_tree_distant_light_importance(
      kg, P, N, kernel_data.integrator.num_distant_lights);
  float light_tree_importance = 0.0f;
  if (kernel_data.integrator.num_distribution > kernel_data.integrator.num_distant_lights) {
    const ccl_global KernelLightTreeNode *kroot = &kernel_data_fetch(light_tree_nodes, 0);
    light_tree_importance = light_tree_cluster_importance(kg, P, N, kroot);
  }
  const float total_group_importance = light_tree_importance + distant_light_importance;
  assert(total_group_importance != 0.0f);
  float pdf = light_tree_importance / total_group_importance;

  const int emitter = (prim >= 0) ? kernel_data_fetch(triangle_to_tree, prim) : kernel_data_fetch(light_to_tree, ~prim);
  ccl_global const KernelLightTreeEmitter* kemitter = &kernel_data_fetch(light_tree_emitters,
                                                                         emitter);
  int parent = kemitter->parent_index;
  ccl_global const KernelLightTreeNode* kleaf = &kernel_data_fetch(light_tree_nodes, parent);

  /* First, we find the probability of selecting the primitive out of the leaf node. */
  float total_importance = 0.0f;
  float emitter_importance = 0.0f;
  for (int i = 0; i < kleaf->num_prims; i++) {
    int prim = i - kleaf->child_index; /* At a leaf node, the negative value is the index into first prim. */
    const float importance = light_tree_emitter_importance(kg, P, N, prim);
    if (i == emitter) {
      emitter_importance = importance;
    }
    total_importance += importance;
  }
  pdf *= emitter_importance / total_importance;
  ...
}
```
We use our newly created arrays to find out the position in the ``light_tree_emitters``, and we then query the ``parent_index`` to find which leaf node contains this emitter. Our leaf node contains all the information we need about the primitives it contains, so we iterate through and find our specific primitive's relative weight. Next, we want to find the probability of actually traversing to this leaf node:
```cpp
ccl_device float light_tree_pdf(KernelGlobals kg, const float3 P, const float3 N, const int prim)
{
  ...
  /* Next, we find the probability of traversing to that leaf node. */
  int child_index = parent; 
  parent = kleaf->parent_index;
  while (parent != -1) {
    const ccl_global KernelLightTreeNode *kparent = &kernel_data_fetch(light_tree_nodes, parent);

    const int left_index = parent + 1;
    const int right_index = kparent->child_index;
    const ccl_global KernelLightTreeNode *kleft = &kernel_data_fetch(light_tree_nodes, left_index);
    const ccl_global KernelLightTreeNode *kright = &kernel_data_fetch(light_tree_nodes, right_index);

    const float left_importance = light_tree_cluster_importance(kg, P, N, kleft);
    const float right_importance = light_tree_cluster_importance(kg, P, N, kright);
    const float left_probability = left_importance / (left_importance + right_importance);

    /* If the child index matches the left index, then we must've traversed left, otherwise right. */
    if (left_index == child_index) {
      pdf *= left_probability;
    }
    else {
      pdf *= (1 - left_probability);
    }

    child_index = parent;
    parent = kparent->parent_index;
  }

  return pdf;
}
```
At any node, we go up to the parent node so that we can compare the children. We also keep track of the child node's index, so we can actually tell whether it was the left or right node. We continue this until our parent index reaches ``-1``, which indicates that we've already hit the root. At this point, we've accounted for everything:
- The probability of sampling the light tree over the distant lights;
- The probability of sampling the specific leaf node containing our primitive;
- The probability of sampling our primitive out of all the primitives in the leaf.
This is the entire process of sampling the light tree, so we can finally return our computed PDF!


### Distant Light Group
The easy case to compute the PDF is the distant lights group. All we have to do is calculate the importance of that specific light, and then divide that by the total importance of the distant lights. We also need to multiply by ``(1 - kernel_data.integrator.pdf_light_tree)``, which is the probability of sampling from the distant light group and not the light tree:
```cpp
// intern\cycles\kernel\light\light_tree.h
ccl_device float distant_lights_pdf(KernelGlobals kg, const float3 P, const float3 N, const int prim)
{
  float distant_light_importance = light_tree_distant_light_importance(
      kg, P, N, kernel_data.integrator.num_distant_lights);
  float light_tree_importance = 0.0f;
  if (kernel_data.integrator.num_distribution > kernel_data.integrator.num_distant_lights) {
    const ccl_global KernelLightTreeNode *kroot = &kernel_data_fetch(light_tree_nodes, 0);
    light_tree_importance = light_tree_cluster_importance(kg, P, N, kroot);
  }
  const float total_group_importance = light_tree_importance + distant_light_importance;
  assert(total_group_importance != 0.0f);
  float pdf = distant_light_importance / total_group_importance;

  /* The light_to_tree array doubles as a lookup table for
   * both the light tree as well as the distant lights group.*/
  const int distant_light = kernel_data_fetch(light_to_tree, prim);
  const int num_distant_lights = kernel_data.integrator.num_distant_lights;

  float emitter_importance = 0.0f;
  float total_importance = 0.0f;
  for (int i = 0; i < num_distant_lights; i++) {
    float importance = light_tree_distant_light_importance(kg, P, N, i);
    if (i == distant_light) {
      emitter_importance = importance;
    }
    total_importance += importance;
  }

  pdf *= emitter_importance / total_importance;
  return pdf;
}
```
The calculation here is pretty straight-forward. Similar to the calculation in the light tree, we cache the emitter's importance while we're computing the total importance.


## Updating MIS in Cycles

Now that we have the code to recalculate probabilities, we're almost done. Just one slight issue: we start at some surface to shoot a ray, but once we've intersected something else and need the MIS calculations, we've lost the normal information of the original surface. To resolve this, we add a struct member called ``mis_origin_n`` to store the normal when we first prepare things for MIS:
```cpp
// intern\cycles\kernel\integrator\shade_surface.h
INTEGRATOR_STATE_WRITE(state, path, mis_ray_pdf) = bsdf_pdf;
INTEGRATOR_STATE_WRITE(state, path, mis_ray_t) = 0.0f;
INTEGRATOR_STATE_WRITE(state, path, mis_origin_n) = sd->N;
INTEGRATOR_STATE_WRITE(state, path, min_ray_pdf) = fminf(
    bsdf_pdf, INTEGRATOR_STATE(state, path, min_ray_pdf));
```
Here, ``sd`` contains information about the current shading point, so we want to hold onto the normal for later. Now we have all the data we need to recalculate our PDFs. In the last post, I mentioned that I was going to be updating some PDF functions, but then I felt like it was actually cleaner to instead update the functions that were actually performing the MIS. These functions included:
- ``integrate_light()`` for light sources
- ``integrate_surface_emission()`` for emissive triangles
- ``integrate_distant_lights()`` for sun lights
- ``integrator_eval_background_shader()`` for background lights
The pattern is pretty much the same for each of these functions. While the MIS weighting is performed, we readjust the ``LightSample``'s PDF to account for the light tree if it's being used:
```cpp
// intern\cycles\kernel\integrator\shade_light.h
/* MIS weighting. */
if (!(path_flag & PATH_RAY_MIS_SKIP)) {
  /* multiple importance sampling, get regular light pdf,
    * and compute weight with respect to BSDF pdf */
  const float mis_ray_pdf = INTEGRATOR_STATE(state, path, mis_ray_pdf);
  if (kernel_data.integrator.use_light_tree) {
    const float3 N = INTEGRATOR_STATE(state, path, mis_origin_n);
    ls.pdf *= light_tree_pdf(kg, ray_P, N, ~ls.lamp);
  }
  const float mis_weight = light_sample_mis_weight_forward(kg, mis_ray_pdf, ls.pdf);
  light_eval *= mis_weight;
}

/* Write to render buffer. */
const float3 throughput = INTEGRATOR_STATE(state, path, throughput);
kernel_accum_emission(kg, state, throughput * light_eval, render_buffer, ls.group);
```
In this case, ``ray_P`` represents the origin of the ray (aka the position of our shading point), but that's something that's already being stored in the integrator state. 


## Closing Thoughts

Overall, this week was a bit heavier on the debugging side again. I think I made too many modifications at the beginning again, which made it a lot harder to debug some things down the line. I had to jump between commits a few times just to make some sanity checks! Thankfully, it seems to have worked out for now...

On the non-MIS side of things, the debugging allowed me to catch some more obvious mistakes which improved the importance heuristics a lot. These things were actually more related to some topics I've already posted about, so I'll try to just edit the existing ones instead of creating new posts. The main contributions were:
- Remodifying some of the triangle sampling logic
- Adjusting the distant light importance heuristic
Sometimes it's a painful process, but as I mentioned before, it's extremely rewarding to see things work out! Again, I'm still using some special test cases, but some scenes are already being rendered in less time (with adaptive sampling) compared to the original method. The last thing I haven't tested yet is environment HDRIs. Hopefully it goes smoothly, but I'm sure I'll run into something that can at least be improved.

