---
layout: post
title: Physically Based Parallax Occlusion Mapping with Self Shadowing
date: 2025-03-17 21:40 +0100
author: bennett
categories: [Computer Graphics & Simulations]
tag: [parallax mapping, hlsl]
math: true
---

This project demonstrates Physically Based Parallax Occlusion Mapping (POM) with self-shadowing in **Unity's Built-In Render Pipeline using HLSL**. It compares three shading models: the Unity Standard Shader, Blinn-Phong (Empirical), and Cook-Torrance with Oren-Nayar (Physically Based). Each of these models also includes a simpler counterpart utilizing basic parallax mapping, highlighting the differences in depth perception and realism.

This page is designed to help solidify one's understanding of parallax mapping and explore the advancements that enhance realism while maintaining a relatively low computational cost.

<p align="center">
  
    <img src="/assets/img/parallax/GIF.gif" alt="Example showcase GIF"/>
  
  <br>
  <em><a href="https://youtu.be/XEO FwgZYHSo"> Watch the showcase on YouTube </a> </em>
</p>

<!-- omit in toc -->
## Table of Contents

- [Overview](#overview)
- [Simple Parallax Mapping](#simple-parallax-mapping)
- [Parallax Occlusion Mapping](#parallax-occlusion-mapping)
- [Self Shadowing](#self-shadowing)
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

A natural way to achieve this scaling effect is by **dividing the $$xy$$ components of $$viewDir$$ by its $$z$$ component**:

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
As you can see the results look decent when looking down at the surface. However, as you might have expected, this basic computation does not come without its drawbacks. When looking at an angle, the algorithm fails to uphold a realistic parallax shift, creating this rather distorted and wavy effect on the plane. 

This happens because our simple parallax mapping only shifts the texture coordinates without actually considering **how different parts of the surface should block or hide others**. In reality, when looking at a rough surface from an angle, some areas should be hidden behind taller parts, while others should be more exposed.

To solve this, we need to look into a more robust approach — **Parallax Occlusion Mapping**.

---
## Parallax Occlusion Mapping


Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed sodales scelerisque risus. Proin ullamcorper cursus arcu, imperdiet semper libero. Sed volutpat ante quis enim elementum, id vulputate quam gravida. Aliquam ullamcorper posuere sapien in dapibus. Proin laoreet odio a nulla fringilla gravida. Quisque vel felis sit amet dui ultricies blandit a eget lectus. Mauris sapien eros, consequat non felis ut, mattis vestibulum mi. Maecenas urna lectus, cursus eget laoreet vel, accumsan molestie mauris. Quisque sed nisl convallis, commodo lectus sit amet, pretium odio. Aenean vitae sapien et enim hendrerit ultricies quis nec ligula. Praesent eu risus nec diam volutpat suscipit.

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
      <td><img src="/assets/img/parallax/POM/BrickBP-UP.jpg" alt="Blinn-Phong"></td>
      <td><img src="/assets/img/parallax/POM/BrickCT-UP.jpg" alt="Cook-Torrance"></td>
    </tr>
  </tbody>
</table>

</details>

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed sodales scelerisque risus. Proin ullamcorper cursus arcu, imperdiet semper libero. Sed volutpat ante quis enim elementum, id vulputate quam gravida. Aliquam ullamcorper posuere sapien in dapibus. Proin laoreet odio a nulla fringilla gravida. Quisque vel felis sit amet dui ultricies blandit a eget lectus. Mauris sapien eros, consequat non felis ut, mattis vestibulum mi. Maecenas urna lectus, cursus eget laoreet vel, accumsan molestie mauris. Quisque sed nisl convallis, commodo lectus sit amet, pretium odio. Aenean vitae sapien et enim hendrerit ultricies quis nec ligula. Praesent eu risus nec diam volutpat suscipit.

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
  </tbody>
</table>
</details>

--- 
## Self Shadowing

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed sodales scelerisque risus. Proin ullamcorper cursus arcu, imperdiet semper libero. Sed volutpat ante quis enim elementum, id vulputate quam gravida. Aliquam ullamcorper posuere sapien in dapibus. Proin laoreet odio a nulla fringilla gravida. Quisque vel felis sit amet dui ultricies blandit a eget lectus. Mauris sapien eros, consequat non felis ut, mattis vestibulum mi. Maecenas urna lectus, cursus eget laoreet vel, accumsan molestie mauris. Quisque sed nisl convallis, commodo lectus sit amet, pretium odio. Aenean vitae sapien et enim hendrerit ultricies quis nec ligula. Praesent eu risus nec diam volutpat suscipit.

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
- [Advanced VR Graphics Techniques](https://developer.arm.com/documentation/102073/0100/Bump-mapping)

## License  

This project is licensed under the **MIT License** – feel free to use, modify, and improve it! 🎨  

