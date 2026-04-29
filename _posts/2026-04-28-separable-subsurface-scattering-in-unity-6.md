---
layout: post
title: Separable Subsurface Scattering in Unity 6
date: 2026-04-28 22:10 +0200
author: bennett
categories: [Computer Graphics]
tags: [subsurface scattering, unity, shader, urp, hlsl]
math: true
image: "assets/img/sss-unity-6/Bust.png"
---

This project is an implementation of Jorge Jimenez's artist-friendly separable model in my [seminar research paper](/research/#real-time-subsurface-scattering-a-comparative-analysis). As a result, this post will solely focus on its implementation in Unity 6 without diving too deep into the theory behind it. 

The repository can be found [here]().

---

## Implementation Overview

![Overview Diagram](../assets/img/sss-unity-6/OverviewDiagram.svg)

Above is the overview of the entire pipeline at a high level which can be separated into three major steps:

1. **Split the lighting**: Separate the material into diffuse, specular, and ambient components.  
2. **Blur the diffuse lighting**: Soften the diffuse part by blurring it horizontally and then vertically to simulate light spreading beneath the surface.  
3. **Combine everything**: Recombine all components to produce the final image.

Now that we have the big picture, let's dive into the details.

---
## 1. Split the lighting

---
## 2. Blur the diffuse lighting
---
## 3. Combine everything
