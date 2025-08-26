---
layout: post
title: 'Forgotten Colors: Capturing Sumi-e in 3D'
date: 2025-08-25 23:31 +0200
author: bennett
categories: [Computer Graphics]
tag: [sumi-e, hlsl, painterly, shader]
math: true
image: "/assets/img/forgotten-colors/house.png"
---

In this post, I share the approaches I explored to achieve the Sumi-e (水墨画) aesthetic for [*Forgotten Colors*](https://felipe-lucas.itch.io/forgotten-colors), a third-person puzzle-platformer where emotions shape reality. Players use talismans to shift the emotional state of each level, transforming both object behaviour and the appearance of the world. Since emotions and atmosphere are central to the experience, the visual style became a crucial part of development.


You can find the source code for the shaders on GitHub [here]().

---

## Motivation and Artistic Direction

At the beginning, we only knew that the game should have a stylised and painterly look, without yet deciding on an oriental influence. As development progressed, sumi-e emerged as a natural fit for the direction we wanted to take. 

Rather than simply copying the traditional monotone sumi-e style, however, our goal was to communicate emotions directly through the levels, which led us to expand the palette with additional colour themes beyond black and white. 

As the sole artist and technical artist on the team, I was responsible for translating this vision into the game’s visuals. This included designing and modelling the main character, creating concept art, and developing shaders to establish the sumi-e look. 

Without a clear starting point, I explored several ideas, starting with adapting a shader I had previously created for my personal 3D render challenge submission [*Boss Fight*](https://youtu.be/TfwCpTyKdX0), by turning the toon shader into a painterly or watercolour style by adjusting the noise and its contrast.

{% include compare-slider.html
   before="/assets/img/forgotten-colors/early-concept-light.png"
   after="/assets/img/forgotten-colors/early-concept-dark.png"
   alt_before="Raw albedo"
   alt_after="Sumi-e shaded"
   start=45
   max_width="900px"
   caption="Early visual exploration of contrasting emotions prior to adopting the sumi-e aesthetic"
%}


---

## First Approach: Reverse-Engineered Lit Shader

![First Approach](/assets/img/forgotten-colors/old%20shader/old-shader.gif)


---

## Second Approach: Unlit Gradients and Stylised Tricks

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Duis eget molestie orci. Aenean non bibendum orci, eu ornare nisl. Orci varius natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. In in ligula ligula. Duis non magna placerat, iaculis nunc bibendum, faucibus mauris. In at urna eget velit convallis venenatis ac eget tellus. Praesent pulvinar nisi purus, quis imperdiet ante volutpat non. Class aptent taciti sociosqu ad litora torquent per conubia nostra, per inceptos himenaeos. Curabitur ut nunc vel magna faucibus feugiat. Orci varius natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Etiam faucibus lacinia ligula, id commodo quam finibus eu. Aliquam ac sollicitudin lorem. Cras lectus dolor, consectetur volutpat velit a, gravida gravida ipsum. Sed ligula felis, egestas sollicitudin tempor non, commodo nec tortor.

---

## Reflections and Next Steps

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Duis eget molestie orci. Aenean non bibendum orci, eu ornare nisl. Orci varius natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. In in ligula ligula. Duis non magna placerat, iaculis nunc bibendum, faucibus mauris. In at urna eget velit convallis venenatis ac eget tellus. Praesent pulvinar nisi purus, quis imperdiet ante volutpat non. Class aptent taciti sociosqu ad litora torquent per conubia nostra, per inceptos himenaeos. Curabitur ut nunc vel magna faucibus feugiat. Orci varius natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Etiam faucibus lacinia ligula, id commodo quam finibus eu. Aliquam ac sollicitudin lorem. Cras lectus dolor, consectetur volutpat velit a, gravida gravida ipsum. Sed ligula felis, egestas sollicitudin tempor non, commodo nec tortor.

---
