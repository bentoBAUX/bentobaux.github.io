---
layout: post
title: 'Forgotten Colors: Capturing Sumi-e in 3D'
date: 2025-08-25 23:31 +0200
author: bennett
categories: [Computer Graphics]
tag: [sumi-e, hlsl, painterly, shader]
math: true
image: "/assets/img/forgotten-colors/previews/house.png"
---

In this post, I share the approaches I explored to achieve the Sumi-e (水墨画) aesthetic for [*Forgotten Colors*](https://felipe-lucas.itch.io/forgotten-colors), a third-person puzzle-platformer where emotions shape reality. Players use talismans to shift the emotional state of each level, transforming both object behaviour and the appearance of the world. Since emotions and atmosphere are central to the experience, the visual style became a crucial part of development.


You can find the source code for the shaders on GitHub [here](https://github.com/bentoBAUX/Sumi-e-in-3D).

---

## Motivation and Artistic Direction

{% include add-image-with-caption.html
   src="/assets/img/forgotten-colors/previews/shozo-sato.png"
   alt="Sumi-e example"
   caption="© 2010 Shozo Sato"
   max_width="1920px"
%}

At the beginning, we only knew that the game should have a stylised and painterly look, without yet deciding on an oriental influence. As development progressed, sumi-e emerged as a natural fit for the direction we wanted to take. 

Rather than simply copying the traditional monotone sumi-e style, however, our goal was to communicate emotions directly through the levels, which led us to expand the palette with additional colour themes beyond black and white. 

As the sole artist and technical artist on the team, I was responsible for translating this vision into the game’s visuals. This included designing and modelling the main character, creating concept art, and developing shaders to establish the sumi-e look. 

Without a clear starting point, I explored several ideas, starting with adapting a shader I had previously created for my personal 3D render challenge submission [***Boss Fight***](https://youtu.be/TfwCpTyKdX0), by turning the toon shader into a painterly or watercolour style by adjusting the noise and its contrast.

{% include compare-slider.html
   before="/assets/img/forgotten-colors/previews/early-concept-light.png"
   after="/assets/img/forgotten-colors/previews/early-concept-dark.png"
   alt_before="Raw albedo"
   alt_after="Sumi-e shaded"
   start=45
   max_width="1920px"
   caption="Early visual exploration of contrasting emotions prior to adopting the sumi-e aesthetic"
%}


---

## First Approach: Reverse-Engineered Lit Shader

<!-- - Talk about the blender shader
- Talk about how i implemented it in hlsl
- Show result
- Explain why it didnt work -->
  
This was my first time developing such a stylised shader for a game in Unity. It meant that the shader had to look good not just on a shader ball, but also when integrated into the actual scene environment.

To explore ideas quickly, I began experimenting in Blender. Playing around with nodes there was much faster and more flexible than constantly rewriting HLSL code. At the same time, I chose to implement the shader in Unity using HLSL because I it is simply more fun for me. 

Working in Blender first also made it clear what was technically possible, which helped me identify when an issue in HLSL was just an implementation mistake rather than a limitation of the approach. So I started by smoothing the noises in the *Boss Fight* shader, adapted from [Kevandram](https://youtu.be/vRALvQSS1vw?t=703). This yielded promising results:

{% include compare-slider.html
   before="/assets/img/forgotten-colors/old%20shader/blender-render.png"
   after="/assets/img/forgotten-colors/old%20shader/blender-nodes.png"
   alt_before="Blender Render"
   alt_after="Blender Nodes"
   start=45
   max_width="1920px"
   caption="Results of the shader experiment in Blender"
%}

Now that I am convinced that it is technically achievable, I began reverse engineering the shader nodes. 

The original Blender graph consisted of five main components: Voronoi Noise, fBM Noise, Principled BSDF, Linear Light and a Colour Ramp. I simplified this to just three parts: Voronoi Noise, Blinn-Phong Lighting, and a Colour Ramp. Despite the reduction, the shader still produces the same overall visual effect:

{% include compare-slider.html
   before="/assets/img/forgotten-colors/old%20shader/unity-render.gif"
   after="/assets/img/forgotten-colors/old%20shader/space-comparison.png"
   alt_before="HLSL Unity Realtime Result"
   alt_after="Comparison"
   start=45
   max_width="1920px"
   caption="HLSL Unity Realtime Result"
%}


Below is the key fragment section: the Voronoi feature points gently disturb the lighting, and the result is remapped through a colour ramp to lock it into those sumi-e style ink bands.

```hlsl
  half4 frag(v2f input) : SV_Target
    {
        // Base surface colour: texture * material tint.
        half4 c = tex2D(_AlbedoTex, input.uv_Albedo) * _DiffuseColour;
        
        // Sampling shadow coords in fragment shader to avoid cascading seams.
        float4 shadowCoords = TransformWorldToShadowCoord(input.fragWorldPos);
        
        // Retrieve the primary directional (or main) light with shadow + distance attenuation.
        Light mainLight = GetMainLight(shadowCoords);

        // Unpacking normals from normal map into world space
        half3 n = ProcessNormals(input);

        // Calculating view vector
        float3 v = normalize(_WorldSpaceCameraPos - input.fragWorldPos);

        // Derive coordinate system for procedural noise
        // "Texture Coordinate" node in Blender for sampling in Generated, Normal, UV, Object space
        half3 noiseCoord = GetTextureSpace(_TextureSpace, n, input.fragLocalPos, input.uv_Albedo);

        float dist; // Voronoi cell distance (unused for now, could drive ink pooling).
        float3 col; // Raw Voronoi cell color (not used directly here).
        float3 pos; // Feature point position (used below to perturb lighting).

        // 3D Smooth F1 Voronoi noise
        // Translated into HLSL from: https://github.com/kinakomoti-321/Voronoi_textures/blob/main/VoronoiTexture/Voronoi.glsl
        VoronoiSmoothF1_3D(noiseCoord * _VoronoiScale, _VoronoiSmoothness, _VoronoiExponent, _VoronoiRandomness, _DistanceMetric, dist, col, pos);

        // Stylised Blinn-Phong lighting
        // Using Voronoi feature point (pos) instead of raw normal injects painterly variation.
        half3 lighting = BlinnPhong(pos, v, mainLight, c) * mainLight.shadowAttenuation * mainLight.distanceAttenuation;

        // Remap lighting through a colour ramp to enforce the sumi-e bands.
        half3 finalColour = ColourRamp(lighting);

        return half4(finalColour, 1.0);
    }
```
<figcaption class="ba-caption">Full Unity shader (fragment + support functions) on GitHub: <a href="https://github.com/bentoBAUX/Sumi-e-in-3D" target="_blank" rel="noopener">View source</a>.</figcaption>

It looked great in isolation, but it fell apart once placed in the actual scene. The method depended heavily on clean topology and UVs, yet most of the level geometry consisted of quick ProBuilder blockouts with awkward topology and unintuitive UV editing. Producing proper assets with retopology, UV packing, and consistent texel density was out of scope while we were also preparing for exams. As a result, the shader required per-object tweaking of every property, where every material relied on a fixed six-step colour ramp that became unmanageable at scene scale.

To achieve the painterly look I could not rely on surface normals, since on large flat faces they all point in the same direction and would have produced flat, uniform colours. This issue was especially pronounced because much of the scene was built from simple, flat shapes. Instead, I used **generated texture coordinates**, similar to Blender’s *Generated* output in their *Texture Coordinate* node, by remapping the object’s local position into the [0,1] range. 

While this gave me the variation I wanted, it also introduced shadow artefacts. On large surfaces, there was incomplete shadow coverage, where small areas near the edges of a shadowed face were still lit even though they should have been completely dark. 

This happened because of the simplification I made earlier. I used the generated coordinates directly into the Voronoi noise and used its position output in place of the surface normal for Blinn–Phong lighting. In other words, the shading was no longer tied to the true surface normal, so the shadow test and the lighting calculation did not align. This is a known side-effect that can even be reproduced in Blender when shading with generated coordinates. 

{% include compare-slider.html
   before="/assets/img/forgotten-colors/old%20shader/shadow-artefact.png"
   after="/assets/img/forgotten-colors/old%20shader/shadow-artefact-blender.png"
   alt_before="Unity"
   alt_after="Blender"
   start=45
   max_width="1920px"
   caption="Incomplete shadowing caused by the generated texture coordinates in Unity (left) and Blender (right)"
%}


To reiterate the tediousness of per-object tweaking, this is an example issue that we had when the level was built quickly with ProBuilder due to time constraints. The auto-generated topology and UVs often produced inconsistent shading, even on flat faces, which meant the same shader settings could look correct on one object and completely off on another.

{% include add-image-with-caption.html
   src="/assets/img/forgotten-colors/old%20shader/cake-artefact.png"
   alt="Unity Render"
   caption="Inconsistencies with flat surfaces belonging to different objects"
   max_width="469px"
%}

Given the few months we had in a semester, this approach, although seemingly viable at first, turned out to be unsustainable. And just like that, I was back at square one, searching for another way to bring sumi-e into the game.

---

## Second Approach: Unlit Gradients and Stylised Tricks

- Explain about Lars mentoring me
- Explain the simple shader
- Showcase

---

## Reflections and Next Steps

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Duis eget molestie orci. Aenean non bibendum orci, eu ornare nisl. Orci varius natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. In in ligula ligula. Duis non magna placerat, iaculis nunc bibendum, faucibus mauris. In at urna eget velit convallis venenatis ac eget tellus. Praesent pulvinar nisi purus, quis imperdiet ante volutpat non. Class aptent taciti sociosqu ad litora torquent per conubia nostra, per inceptos himenaeos. Curabitur ut nunc vel magna faucibus feugiat. Orci varius natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Etiam faucibus lacinia ligula, id commodo quam finibus eu. Aliquam ac sollicitudin lorem. Cras lectus dolor, consectetur volutpat velit a, gravida gravida ipsum. Sed ligula felis, egestas sollicitudin tempor non, commodo nec tortor.

---
