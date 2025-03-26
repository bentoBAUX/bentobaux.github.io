---
layout: post
title: Parallax Mapping with Self Shadowing
date: 2025-03-17 21:40 +0100
author: bennett
categories: [Computer Graphics & Simulations]
tag: [parallax mapping, hlsl]
math: true
---

This project demonstrates Physically Based Parallax Occlusion Mapping (POM) with self-shadowing in **Unity's Built-In Render Pipeline using HLSL**. It compares three shading models: the Unity Standard Shader, Blinn-Phong (Empirical), and Cook-Torrance with Oren-Nayar (Physically Based).

This page is designed to help solidify one's understanding of parallax mapping and explore the advancements that enhance realism while maintaining a relatively low computational cost.

<p align="center">
    <a href="https://youtu.be/XEOFwgZYHSo"><img src="/assets/img/parallax/GIF.gif" alt="Example showcase GIF"/></a>
  <br>
  <em><a href="https://youtu.be/XEOFwgZYHSo">Watch the showcase on YouTube</a></em>
</p>

<!-- omit in toc -->
## Table of Contents

- [Overview](#overview)
- [Simple Parallax Mapping](#simple-parallax-mapping)
- [Steep Parallax Mapping](#steep-parallax-mapping)
- [Parallax Occlusion Mapping](#parallax-occlusion-mapping)
- [Self Shadowing](#self-shadowing)
    - [Understanding Tangent Space and Light Direction](#understanding-tangent-space-and-light-direction)
  - [Shader Parameters](#shader-parameters)
- [Performance Considerations](#performance-considerations)
- [Future Improvements](#future-improvements)
- [Credits](#credits)
- [License](#license)

---

## Overview 

Have you ever seen those mind-bending [optical illusion street art](https://d36tnp772eyphs.cloudfront.net/blogs/1/2019/06/Edgar-Mueller-street-mural-optical-illusion-of-ice-cliff.jpg) that turns a flat sidewalk into a deep dark abyss? From the right angle, it feels like you are standing on the edge of a cliff, staring into the gaping unknown. You might even ask a friend to hold the camera while you carefully step on the painted "debris" to strike a frightened pose for your Instagram. This is the concept behind parallax mapping - shifting textures to trick your eyes into seeing real depth.

This is the concept behind parallax mapping—shifting textures to trick your eyes into perceiving real depth.

---

## Simple Parallax Mapping

At its core, parallax mapping distorts parts of a 2D texture based on how we view the surface. Are we looking straight down at it, or viewing it from an angle? This is called the **view angle**. Another crucial element is determining how much different parts of the texture should shift. This information comes from the **depth map**, which is essentially the inverse of the height map.

With this information, we can now compute the displaced texture coordinates for **each pixel** rendered on the screen.

Firstly, we must **retrieve the depth value** for the current pixel we are computing for. This can be simply done by sampling the depth map with the interpolated UVs for the current pixel. 

>The depth values and depth maps we talk about here are not to be confused with the depth values and depth maps associated with the depth buffer!
{: .prompt-warning}

```hlsl
float2 ParallaxMapping(sampler2D depthMap, float2 texCoords, float3 viewDir, float depthScale)
{
  float depth = tex2D(depthMap, texCoords).r;
}
```
We can now calculate the shift in the form of a vector $$p$$, which depends on the viewing angle and the depth value. The key idea is to **increase the shift when viewing the surface at an angle, and reduce it when looking straight down**. To understand this better, let's analyse how $$viewDir$$ behaves.

<img src="/assets/img/parallax/Visualisation/viewDir_vis.gif" alt="viewDir visualisation GIF"/>


From the visualisation, we can observe the following properties:

- $$viewDir_z$$ approaches **1** when looking directly down at the surface.
- $$viewDir_z$$ approaches **0** when looking at an angle (approaching parallel to the surface).
- Conversely, $$viewDir_y$$ increases as we tilt towards a horizontal view and decreases as we look downward.
- For simplicity, in this visualisation, we are only considering the $$yz$$ plane, so $$viewDir_x$$ is omitted.

This means that **$$viewDir_z$$ effectively acts as a depth factor**, determining how much perspective distortion occurs. When looking straight down, there should be minimal distortion, and when looking at an angle, the shift should be more pronounced.

A natural way to achieve this effect is to **project the 3D view direction onto the 2D texture space** by dividing the $$xy$$ components of $$viewDir$$ by its $$z$$ component:

$$
p = \frac{viewDir_xy}{viewDir_z} * (\text{depth} * \text{depth scale factor})
$$

This division ensures that:

- $$\frac{viewDir_xy}{viewDir_z}$$ is **small** when $$viewDir_z$$ is **large** (looking directly down at the surface).
- $$\frac{viewDir_xy}{viewDir_z}$$ is **large** when $$viewDir_z$$ is **small** (looking at the surface at an angle).

The displacement is then scaled appropriately based on the pixel’s depth value, $$\text{depth}$$, and a user-defined depth scale factor, $$\text{depth scale factor}$$, ensuring a flexible and controllable parallax effect. 

Finally, we compute the new texture coordinates, $$t'$$, by adjusting the texture coordinates, $$t$$:

$$
t' = t - p
$$

This shifts the texture sample position, creating the illusion of depth by making closer regions appear to move more than distant ones.

Here is the implementation of this concept in code:

```hlsl
float2 ParallaxMapping(sampler2D depthMap, float2 texCoords, float3 viewDir, float depthScale)
{
    float depth = tex2D(depthMap, texCoords).r;
    float2 p = viewDir.xy / viewDir.z * depth * depthScale;
    return texCoords - p;
}
```

Here is a visualisation of the approach that I have prepared to ease your understanding. The red ball is the new texel (texture element) that the viewer sees after computing the parallax shift. Notice how the vector $$p$$ (length of the red line) changes as we vary the viewing angle.

<img src="/assets/img/parallax/Visualisation/parallaxmap_vis.gif" alt="Parallax mapping visualisation GIF"/>


Below are the results of this implementation in two different lighting models, **Blinn-Phong** and **Cook-Torrance + Oren-Nayar**. 

<details markdown="1">
  <summary>Expand to view the images</summary>

  <table class="table-custom">
  <thead>
    <tr>
      <th>Blinn-Phong (Empirical)</th>
      <th>Cook-Torrance + Oren-Nayar (Physically Based)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><img src="/assets/img/parallax/Simple/BrickBP-UP.jpg" alt="Blinn-Phong"></td>
      <td><img src="/assets/img/parallax/Simple/BrickCT-UP.jpg" alt="Cook-Torrance"></td>
    </tr>
    <tr>
      <td><img src="/assets/img/parallax/Simple/BrickBP-SIDE.jpg" alt="Blinn-Phong"></td>
      <td><img src="/assets/img/parallax/Simple/BrickCT-SIDE.jpg" alt="Cook-Torrance"></td>
    </tr>
  </tbody>
</table>
</details>


<br>
As you can see the results look decent when looking down at the surface. However, as you might have expected, this basic computation does not come without its drawbacks. When looking at an angle, the algorithm fails to uphold a realistic parallax shift, creating this rather distorted and warped effect on the plane. 

This happens because our simple parallax mapping only shifts the texture coordinates without actually considering **how different parts of the surface should block or hide others**. In reality, when looking at a rough surface from an angle, some areas should be hidden behind taller parts, while others should be more exposed.

To solve this, we need to look into a more robust approach — **Steep Parallax Mapping**.

---
## Steep Parallax Mapping

To address the issue from before and to create a more convincing effect, we need a way to simulate depth more accurately — something that allows us to refine how the surface appears as we look around. Instead of applying a single offset, what if we could trace along the surface in steps, gradually refining the displacement? 

Imagine walking through your room in the dark, trying to avoid bumping into furniture. If you take big steps, you risk misjudging their exact location, possibly stumbling into them, hurting yourself, and waking your parents up. But by taking small, careful steps, you can feel your way around and accurately determine where everything is.

Similarly, Steep Parallax Mapping doesn’t just make one big guess. Instead, it takes multiple smaller steps, gradually refining the depth adjustment to create a much more realistic effect.

Going into more detail, we divide the total depth range (0 to 1) into multiple layers, allowing us to traverse the depth map in smaller, incremental steps. Starting from the viewer’s eye, we step along the view direction ($$viewDir$$) after hitting the surface at point A.

We step into each layer in a fixed ``stepVector``, and at every step, we compare the sampled depth value from the depth map (``currentDepthMapValue``) with the current layer depth (``currentLayerDepth``). If the sampled depth is still greater than the current layer depth, we continue stepping forward until the ``currentDepthMapValue`` is smaller than the ``currentLayerDepth``.

Before we address any questions, let me show you how this can be implemented:

```hlsl
float2 SteepParallaxMapping(sampler2D depthMap, float2 texCoords, float3 viewDir, int numLayers, float depthScale)
{
    // Calculate the size of a single layer
    float layerDepth = 1.0 / numLayers;

    // Determine the step vector based on our view angle
    float2 p = viewDir.xy * depthScale;
    float2 stepVector = p / numLayers;

    // Initialise and set starting values before the loop
    float currentLayerDepth = 0.0;
    float2 currentTexCoords = texCoords;

    // Sample with fixed LOD to avoid issues in loops that will cause GPU timeouts
    float currentDepthMapValue = tex2Dlod(depthMap, float4(currentTexCoords, 0, 0.0)).r;

    // Loop until we have stepped too far below the surface (the surface is where the depth map says it is)
    while (currentLayerDepth < currentDepthMapValue)
    {   
        // Shift the texture coordinates in the direction of step vector
        currentTexCoords -= stepVector;

        // Sample the depthmap with the new texture coordinate to get the new depth value
        currentDepthMapValue = tex2Dlod(depthMap, float4(currentTexCoords, 0, 0.0)).r;

        // Move on to the next layer
        currentLayerDepth += layerDepth;
    }

    return currentTexCoords;
}

```

Why does this work? What does it mean to check if the ``currentLayerDepth`` is bigger than the ``currentDepthMapValue``? The explanation is simpler than you think.

Imagine you're entering a pool in the dark and you want to find the water level. You lower yourself gradually, step by step down the staircase. At each step, you feel if your feet are still in the air or have touched the water. Suddenly in one step, your entire leg is submerged into the water! Here you can conclude for the sake of simplicity that the water level is at the previous step. Don't worry, we will explore a smarter approach later.

Now let's analyse the code. 

<!-- omit in toc -->
**1. Divide the depth map into multiple layers**
```hlsl
float layerDepth = 1.0 / numLayers;
```
We split the total depth range that starts from 0 (top) to 1 (bottom) into evenly spaced layers. Each layer is like a step on the staircase in our pool analogy earlier.


<!-- omit in toc -->
**2. Calculate the step vector**
```hlsl
float2 p = viewDir.xy * depthScale;
float2 stepVector = p / numLayers;
```
Here, we're figuring out how to shift the texture coordinates in the direction we're looking. 
- `viewDir.xy` gives us the viewing angle across the surface.
- `depthScale` controls how exaggerated the depth effect is.
- `p` gives us the total shift in texture coordinates we’ll distribute across the layers
- `stepVector` is the direction in which we step (and also how far) for each layer. Basically determining how large a step in the staircase is in our pool analogy. 

<!-- omit in toc -->
**3. Initialise Values**
```hlsl
float currentLayerDepth = 0.0;
float2 currentTexCoords = texCoords;
float currentDepthMapValue = tex2Dlod(depthMap, float4(currentTexCoords, 0, 0.0)).r;
```
We're just setting our initial values before here we enter the loop. 

`float4 tex2Dlod(sampler2D samp, float4 texcoord)` is used to specify a level of detail (LOD) for texture sampling, so the GPU doesn’t have to figure it out on its own — this helps prevent GPU timeouts.

The LOD is included in the `float4 texcoord` parameter, written like this: `float4(uv.x, uv.y, 0, lod)`.
LOD values start at 0 (full detail), and higher values (1, 2, 3, ...) use lower-resolution mipmap levels.


<!-- omit in toc -->
**4. Step through the layers**
```hlsl
while (currentLayerDepth < currentDepthMapValue)
{   
    currentTexCoords -= stepVector;
    currentDepthMapValue = tex2Dlod(depthMap, float4(currentTexCoords, 0, 0.0)).r;
    currentLayerDepth += layerDepth;
}

return currentTexCoords;
```
We keep marching through the surface layer by layer until we go too deep. Here's what happens at each step:

1. **Check if we're still above the surface.** If our current layer depth is less than the depth from the depth map, it means the surface lies deeper than our current position — in other words, we haven’t reached it yet. So we continue stepping forward.

2. **If we've gone too deep, we stop.** At this point, we return the last texture coordinates where we were still above the surface — this gives us the illusion of correct surface depth.

3. **Otherwise, we keep stepping**:
   - Update `currentTexCoords` by taking the next one in the direction of the `stepVector`. 
   - Sample the depth map at the new texture coordinates. 
   - Proceed to the next layer. 
  
Finally, here are the results of our new approach.

<details markdown="1">
  <summary>Expand to view the images</summary>

  <table class="table-custom">
  <thead>
    <tr>
      <th>Blinn-Phong (Empirical)</th>
      <th>Cook-Torrance + Oren-Nayar (Physically Based)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><img src="/assets/img/parallax/Steep/BrickBP-SIDE.jpg" alt="Blinn-Phong"></td>
      <td><img src="/assets/img/parallax/Steep/BrickCT-SIDE.jpg" alt="Cook-Torrance"></td>
    </tr>
    <tr>
      <td><img src="/assets/img/parallax/Steep/BrickBP-UP.jpg" alt="Blinn-Phong"></td>
      <td><img src="/assets/img/parallax/Steep/BrickCT-UP.jpg" alt="Cook-Torrance"></td>
    </tr>
  </tbody>
</table>
</details>

<br>

This is a huge improvement over the basic version. However, even with a high layer count (256 in this example), you can still notice visible steps or banding between layers. The illusion breaks down slightly because we're still only returning the texture coordinates from the last step before we went too deep, without considering where the actual surface lies between the last two steps.

And beyond visual quality, we also want to minimise the layer count for performance reasons — especially in real-time applications like games.

---
## Parallax Occlusion Mapping

To address both issues that we face in Steep Parallax Mapping, we can take a smarter approach: instead of just stopping at the last valid step, we can **interpolate between the two most recent samples** to more accurately estimate where the surface was intersected.

This is the key idea behind **Parallax Occlusion Mapping** — a refinement of Steep Parallax Mapping that adds this extra step for improved precision and smoother results.

The maths behind it is actually quite simple.

We identify two key moments:

- The last step where we were still **above** the surface

- The first step where we ended up **below** it

At each of these points, we calculate how far we are from the surface. By comparing those offsets, we can estimate how far along that transition the surface was actually crossed.

Thinking of this as,

> *"We're moving from `surfaceOffsetBefore` (above the surface) to `surfaceOffsetAfter` (below the surface). **At what fraction along that path did we hit the surface**?"*

the formula becomes:

$$

\text{weight} = \frac{\text{surfaceOffsetAfter}}{\text{surfaceOffsetAfter} + \text{surfaceOffsetBefore}}

$$

Since because ``surfaceOffsetBefore`` is always [negative](/assets/img/parallax/surfaceOffsetBeforeIsNegative.png), we must flip the positive sign to a negative.

So we compute: 

```hlsl
weight = surfaceOffsetAfter / (surfaceOffsetAfter - surfaceOffsetBefore);
```

This gives us a value between 0 and 1, telling us **how far between the two texture coordinates the surface lies**. We can then use that weight to blend between the texture coordinates from both steps.

Using this, our refined version of the algorithm looks like this:

```hlsl
// Compute the texture coordinates from the previous iteration
float2 prevTexCoords = currentTexCoords + stepVector;

// Compute how far below the surface we are at the current step
float surfaceOffsetAfter = currentDepthMapValue - currentLayerDepth;

// Compute how far above the surface we were at the previous step
float surfaceOffsetBefore = tex2Dlod(depthMap, float4(prevTexCoords, 0, 0)).r - currentLayerDepth + layerDepth;

// Linearly interpolate the texture coordinates using the calculated weight
float weight = surfaceOffsetAfter / (surfaceOffsetAfter - surfaceOffsetBefore);
float2 finalTexCoords = prevTexCoords * weight + currentTexCoords * (1.0 - weight);

return finalTexCoords;
```

And here are the results without the artefacts from before with the same layer count:

<details markdown="1">
  <summary>Expand to view the images</summary>

  <table class="table-custom">
  <thead>
    <tr>
      <th>Blinn-Phong (Empirical)</th>
      <th>Cook-Torrance + Oren-Nayar (Physically Based)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><img src="/assets/img/parallax/POM/BrickBP-SIDE.jpg" alt="Blinn-Phong"></td>
      <td><img src="/assets/img/parallax/POM/BrickCT-SIDE.jpg" alt="Cook-Torrance"></td>
    </tr>
    <tr>
      <td><img src="/assets/img/parallax/POM/BrickBP-UP.jpg" alt="Blinn-Phong"></td>
      <td><img src="/assets/img/parallax/POM/BrickCT-UP.jpg" alt="Cook-Torrance"></td>
    </tr>
  </tbody>
</table>
</details>


--- 
## Self Shadowing

We have achieved what we sought to achieve - a relatively low-cost approach in enhancing realism by faking depth on a flat surface. 

Why don't we take it a step further?

If you set up your scene with lights, you will notice that even with convincing surface detail, the illusion breaks down when lighting interacts with the surface. If light hits a deep groove or crack, we expect parts of it to be in shadow — yet everything looks uniformly lit and flat. 

To make our illusion even more convincing, we need to simulate this interaction between light and surface detail. 

This is where **self-shadowing** comes into play.

#### Understanding Tangent Space and Light Direction

Before we can simulate self-shadowing, it’s important to understand why we work in tangent space and how light vectors behave in this local coordinate system.

**Why Tangent Space?**

Our height maps, normal maps, and UVs are defined in texture space, but light and view directions exist in world space. To make lighting interact correctly with texture-based surface detail, we transform those vectors into tangent space — a local space aligned with each fragment’s surface and UV layout.

Tangent space makes lighting calculations independent of the object’s rotation. Without it, the effect would look visually incorrect — shadows and highlights would stay fixed relative to world space instead of following the surface. Rotating the object would break the illusion, as lighting would no longer match the texture detail.

**How Light Vectors Behave In Tangent Space**

In tangent space, the surface is aligned like this:

- +X → tangent (U direction)

- +Y → bitangent (V direction)

- +Z → surface normal, pointing outward

So a light shining from in front of the surface will point toward −Z in tangent space. This is why the Z component of the light vector becomes so important for self-shadowing.

**What to Do With This Information**

Now that we know how to interpret light direction in tangent space, we can use the Z component to decide whether self-shadowing should be calculated at all.

If `lightDir.z >= 0`, the light is behind the surface, and no surface detail should cast shadows in this case. To avoid unnecessary computation, we simply skip the shadowing pass:

```hlsl
if (lightDir.z >= 0.0) return 0.0;
```

This keeps the shader efficient and ensures that shadow rays are only traced when the surface is lit from the front — where shadowing is visually meaningful.

**Self Shadowing Algorithm**

Once we've confirmed that the light is in front of the surface (`lightDir.z < 0.0`), we can proceed with simulating how light rays interact with the surface detail described by the height map. The idea is to march along the light's direction in tangent space, checking if any elevated point along the path blocks the light.

First, we calculate the number of layers and the step size per layer:

```hlsl
float layerDepth = 1.0 / numLayers;
vec2 p = lightDir.xy / lightDir.z * depthScale;
vec2 stepVector = p / numLayers;
```

Here, `p` is the total projected UV offset in the direction of the light, scaled by `depthScale`. Dividing by `numLayers` gives us the UV step for each iteration.

We then begin ray marching from the current texture coordinates and initial depth:

```hlsl
vec2 currentTexCoords = texCoords;
float currentDepthMapValue = tex2D(depthMap, currentTexCoords).r;
float currentLayerDepth = currentDepthMapValue;
```

To avoid false shadowing from very small height changes, we introduce a small bias:

```hlsl
float shadowBias = 0.03;
```
This helps prevent high-frequency shadow noise by allowing shallow indentations to remain lit. We also limit the number of iterations to keep the shader efficient and compatible with GPU loop unrolling limits:

```hlsl
int maxIterations = 32;
int iterationCount = 0;
```

Now we walk along the light direction, layer by layer:

```hlsl
while (currentLayerDepth <= currentDepthMapValue + shadowBias && currentLayerDepth > 0.0 && iterationCount < maxIterations)
{
    currentTexCoords += stepVector;
    currentDepthMapValue = tex2D(depthMap, currentTexCoords).r;
    currentLayerDepth -= layerDepth;
    iterationCount++;
}
```
This loop ends if the ray exits the surface, hits an occluder, or exceeds the iteration limit.

Finally, we determine if the light path is blocked:

```hlsl
return currentLayerDepth > currentDepthMapValue ? 0.0 : 1.0;
```

If the ray reached a depth where the height map is still lower, the fragment is shadowed (returns 0.0). Otherwise, it’s fully lit (1.0).

Here are the final results for this tutorial:

<details markdown="1">
  <summary>Expand to view the images</summary>

  <table class="table-custom">
  <thead>
    <tr>
      <th>Blinn-Phong (Empirical)</th>
      <th>Cook-Torrance + Oren-Nayar (Physically Based)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><img src="/assets/img/parallax/SelfShadow/BrickBP-UP.jpg" alt="Blinn-Phong"></td>
      <td><img src="/assets/img/parallax/SelfShadow/BrickCT-UP.jpg" alt="Cook-Torrance"></td>
    </tr>
    <tr>
      <td><img src="/assets/img/parallax/SelfShadow/BrickBP-SIDE.jpg" alt="Blinn-Phong"></td>
      <td><img src="/assets/img/parallax/SelfShadow/BrickCT-SIDE.jpg" alt="Cook-Torrance"></td>
    </tr>
  </tbody>
</table>

</details>

---


### Shader Parameters  

| **Parameter**     | **Description**                                |
| ----------------- | ---------------------------------------------- |
| `_HeightScale`    | Controls the depth intensity of POM            |
| `_NumberOfLayers` | Adjusts the precision of parallax calculation  |
| `_Metallic`       | Controls how metallic the surface appears      |
| `_Roughness`      | Affects surface roughness and light scattering |
| `_NormalStrength` | Strength of normal map details                 |

---
## Performance Considerations  

- **Use lower `NumberOfLayers` for better performance.**  
- **Steep angles require more samples; consider LOD adjustments.**  
- **Avoid overusing self-shadowing on high-performance constraints.**  

## Future Improvements  

- Add support for **dynamic tessellation**.  
- Improve self-shadowing accuracy for extreme angles.  
- Optimize performance with **adaptive sampling techniques**.  


## Credits  

- Joey de Vries for his insightful tutorial on [LearnOpenGL](https://learnopengl.com/Advanced-Lighting/Parallax-Mapping), which provided a strong foundation for this project’s development.
- Rabbid76 on [StackOverflow](https://stackoverflow.com/questions/55089830/adding-shadows-to-parallax-occlusion-map) for the tutorial on self shadowing for parallax mapping.

## License  

This project is licensed under the **MIT License** – feel free to use, modify, and improve it! 🎨  

