---
layout: page
permalink: /research/
icon: fas fa-book
order: 1
toc: true
---
---
## Real-Time Subsurface Scattering: A Comparative Analysis

<div align="center">
  <img src="/assets/research/realtime-subsurface-scattering/thumbnail.png" alt="Realtime SSS Thumbnail" width="100%">
</div>  

#### Summary
This paper examines three real-time subsurface scattering techniques—Simple Translucency Approximations, Separable Subsurface Scattering, and Christensen–Burley’s Normalised Diffusion—analysing their underlying principles, visual characteristics, and trade-offs in real-time rendering.

#### Key Insights
- The screen space implementation of Separable SSS and Christensen-Burley’s Normalised Diffusion exhibit similar computational complexity.
- Simple translucency approximation remains significantly cheaper but fails to capture realistic multi-scattering behaviour.
  
#### Practical Note
In practice, both Separable SSS and Christensen–Burley are implemented as screen-space diffusion approximations. They do not model true subsurface light transport or backlighting, but instead blur rendered surface radiance.

**Type:** Seminar Paper (TUM)  
**Year:** 2026

<p>
  <a href="{{ '/assets/research/realtime-subsurface-scattering/paper.pdf' | relative_url }}">Read Paper (PDF)</a>
  &nbsp;&nbsp;&nbsp;
  <a href="{{ '/assets/research/realtime-subsurface-scattering/slides.pdf' | relative_url }}">View Slides (PDF)</a>
</p>

---
