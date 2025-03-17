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
  <a href="https://youtu.be/XEOFwgZYHSo">
    <img src="/assets/img/parallax/GIF.gif" alt="Example showcase GIF" />
  </a>
  <br>
  <em>Click to watch the showcase on YouTube</em>
</p>

## Table of Contents

- [Table of Contents](#table-of-contents)
- [Parallax Mapping](#parallax-mapping)
  - [How It Works](#how-it-works)
- [Parallax Occlusion Mapping](#parallax-occlusion-mapping)
  - [How It Works](#how-it-works-1)
- [Self Shadowing](#self-shadowing)
  - [How It Works](#how-it-works-2)
  - [Shader Parameters](#shader-parameters)
- [Performance Considerations](#performance-considerations)
- [Future Improvements](#future-improvements)
- [Credits](#credits)
- [License](#license)

## Parallax Mapping

### How It Works  

Have you ever seen those mind-bending [optical illusion street art](https://d36tnp772eyphs.cloudfront.net/blogs/1/2019/06/Edgar-Mueller-street-mural-optical-illusion-of-ice-cliff.jpg) that turns a flat sidewalk into a deep dark abyss? From the right angle, it feels like you are standing on the edge of a cliff, staring into the gaping unknown. You might even ask a friend to hold the camera while you carefully step on the painted "debris" to strike a frightened pose for your Instagram. This is the concept behind parallax mapping - shifting textures to trick your eyes into seeing real depth.

As mentioned, parallax mapping's core concept is distorting parts of the 2D texture applied to a surface based on how we are looking at it. This means we need to consider _how_ we are looking at the surface - are we staring straight down at it, or viewing it from an angle? This would be our **view angle**. We also need to know how much we would like a part of the texture to shift. In our case, we’ll be using a **depth map**, which is simply an inverse of the heightmap.



<details markdown="1">
  <summary>Expand to view the images</summary>

  | **Blinn-Phong (Empirical)**                                                | **Cook-Torrance + Oren-Nayar (Physically Based)**                          |
  | -------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
  | <img src="/assets/img/parallax/Simple/BrickBP-UP.jpg" width="800px"> | <img src="/assets/img/parallax/Simple/BrickCT-UP.jpg" width="800px"> |

</details>

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed sodales scelerisque risus. Proin ullamcorper cursus arcu, imperdiet semper libero. Sed volutpat ante quis enim elementum, id vulputate quam gravida. Aliquam ullamcorper posuere sapien in dapibus. Proin laoreet odio a nulla fringilla gravida. Quisque vel felis sit amet dui ultricies blandit a eget lectus. Mauris sapien eros, consequat non felis ut, mattis vestibulum mi. Maecenas urna lectus, cursus eget laoreet vel, accumsan molestie mauris. Quisque sed nisl convallis, commodo lectus sit amet, pretium odio. Aenean vitae sapien et enim hendrerit ultricies quis nec ligula. Praesent eu risus nec diam volutpat suscipit.

<details markdown="1">
  <summary>Expand to view the images</summary>

  | **Blinn-Phong (Empirical)**                                                  | **Cook-Torrance + Oren-Nayar (Physically Based)**                            |
  | ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
  | <img src="/assets/img/parallax/Simple/BrickBP-SIDE.jpg" width="800px"> | <img src="/assets/img/parallax/Simple/BrickCT-SIDE.jpg" width="800px"> |

</details>


---
## Parallax Occlusion Mapping

### How It Works  

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed sodales scelerisque risus. Proin ullamcorper cursus arcu, imperdiet semper libero. Sed volutpat ante quis enim elementum, id vulputate quam gravida. Aliquam ullamcorper posuere sapien in dapibus. Proin laoreet odio a nulla fringilla gravida. Quisque vel felis sit amet dui ultricies blandit a eget lectus. Mauris sapien eros, consequat non felis ut, mattis vestibulum mi. Maecenas urna lectus, cursus eget laoreet vel, accumsan molestie mauris. Quisque sed nisl convallis, commodo lectus sit amet, pretium odio. Aenean vitae sapien et enim hendrerit ultricies quis nec ligula. Praesent eu risus nec diam volutpat suscipit.

<details markdown="1">
  <summary>Expand to view the images</summary>

  | **Blinn-Phong (Empirical)**                                                | **Cook-Torrance + Oren-Nayar (Physically Based)**                          |
  | -------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
  | <img src="/assets/img/parallax/POM/BrickBP-UP.jpg" width="800px"> | <img src="/assets/img/parallax/POM/BrickCT-UP.jpg" width="800px"> |

</details>

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed sodales scelerisque risus. Proin ullamcorper cursus arcu, imperdiet semper libero. Sed volutpat ante quis enim elementum, id vulputate quam gravida. Aliquam ullamcorper posuere sapien in dapibus. Proin laoreet odio a nulla fringilla gravida. Quisque vel felis sit amet dui ultricies blandit a eget lectus. Mauris sapien eros, consequat non felis ut, mattis vestibulum mi. Maecenas urna lectus, cursus eget laoreet vel, accumsan molestie mauris. Quisque sed nisl convallis, commodo lectus sit amet, pretium odio. Aenean vitae sapien et enim hendrerit ultricies quis nec ligula. Praesent eu risus nec diam volutpat suscipit.

<details markdown="1">
  <summary>Expand to view the images</summary>

  | **Blinn-Phong (Empirical)**                                                | **Cook-Torrance + Oren-Nayar (Physically Based)**                          |
  | -------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
  | <img src="/assets/img/parallax/POM/BrickBP-SIDE.jpg" width="800px"> | <img src="/assets/img/parallax/POM/BrickCT-SIDE.jpg" width="800px"> |

</details>

--- 
## Self Shadowing

### How It Works  

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed sodales scelerisque risus. Proin ullamcorper cursus arcu, imperdiet semper libero. Sed volutpat ante quis enim elementum, id vulputate quam gravida. Aliquam ullamcorper posuere sapien in dapibus. Proin laoreet odio a nulla fringilla gravida. Quisque vel felis sit amet dui ultricies blandit a eget lectus. Mauris sapien eros, consequat non felis ut, mattis vestibulum mi. Maecenas urna lectus, cursus eget laoreet vel, accumsan molestie mauris. Quisque sed nisl convallis, commodo lectus sit amet, pretium odio. Aenean vitae sapien et enim hendrerit ultricies quis nec ligula. Praesent eu risus nec diam volutpat suscipit.

<details markdown="1">
  <summary>Expand to view the images</summary>

  | **Blinn-Phong (Empirical)**                                                | **Cook-Torrance + Oren-Nayar (Physically Based)**                          |
  | -------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
  | <img src="/assets/img/parallax/SelfShadow/BrickBP-UP.jpg" width="800px"> | <img src="/assets/img/parallax/SelfShadow/BrickCT-UP.jpg" width="800px"> |

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

