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

This project is an implementation of Jorge Jimenez's artist-friendly separable model in my [**seminar research paper**](/research/#real-time-subsurface-scattering-a-comparative-analysis). As a result, this post will solely focus on its implementation in Unity 6 without diving too deep into the theory behind it. 

The repository can be found [**here**]().

---

## Implementation Overview

![Overview Diagram](../assets/img/sss-unity-6/OverviewDiagram.svg)

Above is the overview of the entire pipeline at a high level which can be separated into three major steps:

1. **Split the lighting into separate buffers**: Separate the material into diffuse, specular, and ambient components.  
2. **Blur the diffuse lighting**: Soften the diffuse part by blurring it horizontally and then vertically to simulate light spreading beneath the surface.  
3. **Combine everything**: Recombine all components to produce the final image.

Now that we have the big picture, let's dive into the details.

---
## 1. Split the lighting into separate buffers
Our goal in this step is to neatly separate the diffuse, specular, and ambient components of the lighting for the next stage of our pipeline. This step is necessary since the subsurface scattering pass will only operate on the diffuse part of the lighting.

> Why diffuse part only?
> {: .title}
>
> In a normal diffuse lighting model, light is treated as if it enters and exits at the same surface point.
> Screen-space SSS modifies this by spreading the diffuse lighting into neighbouring pixels, creating the impression that light travelled underneath the surface.
> 
> Specular lighting is kept sharp because it represents direct surface reflection. Ambient lighting is kept separate so it can be added back in a controlled way after the SSS blur.
{: .box-info}

Normally, a fragment shader writes one final colour output to a single **colour target**. A colour target is a GPU-side destination that receives the colour values produced by the shader.

In HLSL, this is done through the [**`SV_Target`**](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics#system-value-semantics) semantic. `SV_Target0` refers to the first colour target of the current render pass. In Unity, that colour target is often a [**`RenderTexture`**](https://docs.unity3d.com/2022.2/Documentation/Manual/class-RenderTexture.html), allowing the rendered result to be reused by later shader passes.

For multiple render targets, we simply write to more outputs: `SV_Target1`, `SV_Target2`, and so on. Unity supports up to 8 colour targets in total, from `SV_Target0` to `SV_Target7`. In this implementation, we only need the first three for each of the lighting components.

Using [**this**]() PBR shader as a starting point, we can refactor the fragment output so each lighting component is written to its own colour target. 


>Since this shader supports multiple lighting models, each with its own settings, I use a custom `ShaderGUI` to keep the material inspector clean and only show the controls relevant to the selected model. The script for this can be found [**here**]().
{:.prompt-info}

#### 1.1 Refactoring the fragment output

A standard lit fragment shader in HLSL usually returns one final colour:

```hlsl
float4 frag(Varyings IN) : SV_Target0
{
  ...
  float3 lighting = LightLoop(surfaceData, inputData);
  return float4(finalColour, 1);
}

```
Here, `LightLoop()` calculates the lighting using two structs: `inputData` and `surfaceData`.

`inputData` is mainly required by URP’s lighting system, especially the Forward+ light loop. `surfaceData` is our own shader data struct, used to store the material information needed by our lighting functions. There is some overlap between them, such as position, normal, and view direction. That is fine: `inputData` is for URP, while `surfaceData` is for our own lighting code.

The shader currently writes the final `float4` colour value into `SV_Target0` after all lighting calculations are complete. To split this up, we can define a custom fragment output `struct` with each field mapped to a different colour target:

```hlsl
// Define this before your fragment shader
struct FragOutput
{
  float4 diffuseBuffer: SV_Target0;
  float4 specularBuffer: SV_Target1;
  float4 ambientBuffer : SV_Target2;
};
```
We can then rewrite our fragment shader and `LightLoop()` to return a `FragOutput` instead of a `float4`:

```hlsl
FragOutput frag(Varyings IN)
{
  ...
  FragOutput o = LightLoop(surfaceData, inputData);
  return o;
}
```

The same refactor also has to happen inside `LightLoop()` where each component of the lighting is accumulated in separate buckets. For the full shader changes, compare [**`SSSS Master.shader`**](...) with [**`PBR Master.shader`**](...).

#### 1.2 Creating the lighting buffers

By using the `SV_Target[i]` semantics, we have only labelled where each output should be written.

So far, only our diffuse output will render because `SV_Target0` is usually bound to the camera colour target in normal render passes. Since nothing stores the other specular and ambient lighting information, the other two colour targets won't be rendered.

To fix this, we must create our own custom render pass that properly receives all three outputs as separate render textures. This can be done in Unity 6 by creating a custom [**`ScriptableRenderFeature`**](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@17.3/api/UnityEngine.Rendering.Universal.ScriptableRendererFeature.html)(SRF) with a [**`ScriptableRenderPass`**](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@17.3/api/UnityEngine.Rendering.Universal.ScriptableRenderPass.html). An SRF is a component that can be added to a scriptable renderer like URP to modify how the scene is rendered. It configures and enqueues render passes that contain the actual rendering work. 

> How to remember the difference?
> {: .title}
> 
> `ScriptableRenderFeature` tells URP: *“Please insert this custom rendering operation into the renderer.”*
> 
> `ScriptableRenderPass` tells URP: *“Here is what to do when that operation runs.”*
{: .box-tip}

#### 1.3 Setting up the render feature

In our case, we want a render feature that tells URP to insert our custom lighting-buffer pipeline into the renderer. This pipeline captures the diffuse, specular, and ambient lighting buffers, processes the diffuse buffer for subsurface scattering, and then combines everything back into the final image.

The `ScriptableRenderFeature` itself is mostly boilerplate. It simply creates our custom render pass and inserts it into URP. For the basic steps of adding a render feature to URP, see [this](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@16.0/manual/urp-renderer-feature-how-to-add.html). 


<details class="collapsible" markdown="1">
<summary>
  <span class="collapsible-label">Show:</span>
  <span class="collapsible-meta"><code>SSSSRenderFeature.cs</code></span>
</summary>

```csharp
using UnityEngine;
using UnityEngine.Rendering.Universal;

public class SSSSRenderFeature : ScriptableRendererFeature
{
    [SerializeField] private Material SSSSMaterial;
    [SerializeField] private Material compositeMaterial;

    private SSSSRenderPass renderPass;

    public override void Create()
    {
        renderPass = new SSSSRenderPass(SSSSMaterial, compositeMaterial);
        renderPass.renderPassEvent = RenderPassEvent.AfterRenderingOpaques;
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (renderPass == null)
        {
            Debug.LogError("SSSSRenderFeature: RenderPass was not created.");
            return;
        }

        renderer.EnqueuePass(renderPass);
    }

    protected override void Dispose(bool disposing)
    {
        if (disposing && renderPass != null)
            renderPass.Dispose();
    }
}
```
</details>

Nothing particularly SSSS-specific happens inside the `ScriptableRenderFeature`, so we can move on quickly.

The important part is the `ScriptableRenderPass`. This is where we actually create the lighting buffers, bind them as render targets, run the blur passes, and composite the final image.

#### 1.4 Building the render pass

In Unity 6, the `ScriptableRenderPass` workflow has changed slightly. In older URP versions, a custom pass usually meant giving Unity a sequence of rendering commands and manually managing temporary render textures yourself.

With RenderGraph, we instead describe the pass through `RecordRenderGraph()`. Each pass declares which textures it reads from and writes to, allowing Unity to manage resource lifetimes, pass ordering, and optimisation more safely.

For our SSS pipeline, the render pass has **five main parts**:

1. Create temporary lighting buffers.
2. Render the SSS objects into separate diffuse, specular, and ambient textures.
3. Blur the diffuse texture.
   1. Blur the diffuse texture horizontally.
   2. Blur the result vertically.
4. Composite the blurred diffuse lighting with the other lighting components.
5. Blitt the final texture into the scene texture.

A line-by-line walkthrough would make this devlog too long, so I’ve included the script below with the key sections commented for clarity.

<details class="collapsible" markdown="1">
<summary>
  <span class="collapsible-label">Show:</span>
  <span class="collapsible-meta"><code>SSSSRenderPass.cs</code></span>
</summary>

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.RenderGraphModule;
using UnityEngine.Rendering.Universal;

public class SSSSRenderPass : ScriptableRenderPass
{
    private readonly Material compositeMaterial;
    private readonly Material SSSSMaterial;

    // Stores MRT textures so the later composite pass can read them in the same frame.
    private class SSSSFrameData : ContextItem
    {
        public TextureHandle diffuseTexture;
        public TextureHandle processedDiffuseTexture;
        public TextureHandle specularTexture;
        public TextureHandle ambientTexture;

        public override void Reset()
        {
            diffuseTexture = TextureHandle.nullHandle;
            specularTexture = TextureHandle.nullHandle;
            ambientTexture = TextureHandle.nullHandle;
            processedDiffuseTexture = TextureHandle.nullHandle;
        }
    }

    // Data containers used by the individual RenderGraph passes.
    // These carry textures, materials, or renderer lists from pass setup into execution.

    private class MRTPassData
    {
        public RendererListHandle rendererList;
    }

    private class SSSSPassData
    {
        public TextureHandle inputTexture;
        public Material material;
    }

    private class CompositePassData
    {
        public TextureHandle sceneTexture;
        public TextureHandle diffuseTexture;
        public TextureHandle processedDiffuseTexture;
        public TextureHandle specularTexture;
        public TextureHandle ambientTexture;
        public Material compositeMaterial;
    }

    private class FinalBlitPassData
    {
        public TextureHandle sourceTexture;
        public Material blitMaterial;
    }

    public SSSSRenderPass(Material SSSSMaterial, Material compositeMaterial)
    {
        this.SSSSMaterial = SSSSMaterial;
        this.compositeMaterial = compositeMaterial;
    }

    public override void RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData)
    {
        if (compositeMaterial == null)
        {
            Debug.LogError("compositeMaterial is null");
            return;
        }

        if (SSSSMaterial == null)
        {
            Debug.LogError("SSSSMaterial is null");
            return;
        }

        UniversalRenderingData renderingData = frameData.Get<UniversalRenderingData>();
        UniversalCameraData cameraData = frameData.Get<UniversalCameraData>();
        UniversalLightData lightData = frameData.Get<UniversalLightData>();
        UniversalResourceData resourceData = frameData.Get<UniversalResourceData>();

        // 1) Setup: Create MRT textures matching the camera target.
        RenderTextureDescriptor desc = cameraData.cameraTargetDescriptor;
        desc.depthBufferBits = 0;
        desc.msaaSamples = 1;

        TextureHandle ambientTexture = UniversalRenderer.CreateRenderGraphTexture(renderGraph, desc, "_SSSSAmbientTexture", false);
        TextureHandle diffuseTexture = UniversalRenderer.CreateRenderGraphTexture(renderGraph, desc, "_SSSSDiffuseTexture", false);
        TextureHandle horizontalTempTexture = UniversalRenderer.CreateRenderGraphTexture(renderGraph, desc, "_SSSSHorizontalTempTexture", false);
        TextureHandle processedDiffuseTexture = UniversalRenderer.CreateRenderGraphTexture(renderGraph, desc, "_SSSSProcessedDiffuseTexture", false);
        TextureHandle specularTexture = UniversalRenderer.CreateRenderGraphTexture(renderGraph, desc, "_SSSSSpecularTexture", false);
        TextureHandle compositeOutputTexture = UniversalRenderer.CreateRenderGraphTexture(renderGraph, desc, "_SSSSCompositeOutputTexture", false);

        // Make them available to later passes in this frame.
        SSSSFrameData customData = frameData.Create<SSSSFrameData>();
        customData.diffuseTexture = diffuseTexture;
        customData.processedDiffuseTexture = processedDiffuseTexture;
        customData.specularTexture = specularTexture;
        customData.ambientTexture = ambientTexture;

        // 2) MRT pass: attach ambient, diffuse, specular into separate render textures for SSS processing
        using (var builder = renderGraph.AddRasterRenderPass<MRTPassData>("SSSS MRT Pass", out var passData))
        {
            ShaderTagId shaderTag = new ShaderTagId("UniversalForward");

            SortingCriteria sortFlags = cameraData.defaultOpaqueSortFlags;
            FilteringSettings filteringSettings = new FilteringSettings(RenderQueueRange.opaque, LayerMask.GetMask("SSSS"));

            DrawingSettings drawingSettings = RenderingUtils.CreateDrawingSettings(
                shaderTag,
                renderingData,
                cameraData,
                lightData,
                sortFlags
            );

            RendererListParams rendererListParams = new RendererListParams(
                renderingData.cullResults,
                drawingSettings,
                filteringSettings
            );

            // Create the renderer list, store it in passData, then register it with the builder.
            passData.rendererList = renderGraph.CreateRendererList(rendererListParams);
            builder.UseRendererList(passData.rendererList);

            // MRT bindings:
            // SV_Target0 -> diffuseTexture
            // SV_Target1 -> specularTexture
            // SV_Target2 -> ambientTexture
            builder.SetRenderAttachment(diffuseTexture, 0);
            builder.SetRenderAttachment(specularTexture, 1);
            builder.SetRenderAttachment(ambientTexture, 2);

            // Use the active depth buffer so geometry depth-tests correctly.
            builder.SetRenderAttachmentDepth(resourceData.activeDepthTexture, AccessFlags.ReadWrite);

            // Keep this on while debugging so RenderGraph does not cull the pass away.
            builder.AllowPassCulling(false);

            // Define the actual GPU commands for this pass.
            // The MRTPassData filled above is received here as 'data'.
            builder.SetRenderFunc(static (MRTPassData data, RasterGraphContext context) =>
            {
                context.cmd.ClearRenderTarget(false, true, Color.black);
                context.cmd.DrawRendererList(data.rendererList);
            });
        }

        // 3a) SSS horizontal pass: Apply horizontal SSS pass onto diffuse texture and output it to horizontalTempTexture.
        using (var builder = renderGraph.AddRasterRenderPass<SSSSPassData>("SSSS SSS Horizontal Pass", out var passData))
        {
            // Setup the data needed when the pass executes.
            passData.inputTexture = diffuseTexture;
            passData.material = SSSSMaterial;

            // Declare the texture read and output target.
            builder.UseTexture(passData.inputTexture);
            builder.SetRenderAttachment(horizontalTempTexture, 0);

            builder.AllowPassCulling(false);

            // Execute material pass 0, which performs the horizontal SSS blur.
            builder.SetRenderFunc(static (SSSSPassData data, RasterGraphContext context) =>
            {
                data.material.SetTexture("_MainTex", data.inputTexture);
                Blitter.BlitTexture(context.cmd, data.inputTexture, new Vector4(1, 1, 0, 0), data.material, 0);
            });
        }

        // 3b) SSS vertical pass: Apply vertical SSS pass onto 3a's texture and output it to processedDiffuseTexture.
        using (var builder = renderGraph.AddRasterRenderPass<SSSSPassData>("SSSS SSS Vertical Pass", out var passData))
        {
            passData.inputTexture = horizontalTempTexture;
            passData.material = SSSSMaterial;

            builder.UseTexture(passData.inputTexture);
            builder.SetRenderAttachment(processedDiffuseTexture, 0);
            builder.AllowPassCulling(false);

            builder.SetRenderFunc(static (SSSSPassData data, RasterGraphContext context) =>
            {
                data.material.SetTexture("_MainTex", data.inputTexture);
                Blitter.BlitTexture(context.cmd, data.inputTexture, new Vector4(1, 1, 0, 0), data.material, 1);
            });
        }

        // 4) Composite pass: Combine processedDiffuseTexture with specularTexture and apply the masked result into compositeOutputTexture
        using (var builder = renderGraph.AddRasterRenderPass<CompositePassData>("SSSS Composite Pass", out var passData))
        {
            // Retrieve the textures created earlier in this frame.
            // These were written by the MRT and blur passes.
            SSSSFrameData custom = frameData.Get<SSSSFrameData>();

            // Data needed when the composite pass executes.
            passData.sceneTexture = resourceData.activeColorTexture;
            passData.diffuseTexture = custom.diffuseTexture;
            passData.processedDiffuseTexture = custom.processedDiffuseTexture;
            passData.specularTexture = custom.specularTexture;
            passData.ambientTexture = custom.ambientTexture;
            passData.compositeMaterial = compositeMaterial;

            // Declare all texture reads.
            builder.UseTexture(passData.sceneTexture);
            builder.UseTexture(passData.diffuseTexture);
            builder.UseTexture(passData.processedDiffuseTexture);
            builder.UseTexture(passData.specularTexture);
            builder.UseTexture(passData.ambientTexture);

            // Store the composited result in compositeOutputTexture.
            builder.SetRenderAttachment(compositeOutputTexture, 0);

            builder.AllowPassCulling(false);

            // Run composite material pass 0.
            builder.SetRenderFunc(static (CompositePassData data, RasterGraphContext context) =>
            {
                data.compositeMaterial.SetTexture("_SceneTex", data.sceneTexture);
                data.compositeMaterial.SetTexture("_DiffuseTex", data.diffuseTexture);
                data.compositeMaterial.SetTexture("_ProcessedDiffuseTex", data.processedDiffuseTexture);
                data.compositeMaterial.SetTexture("_SpecularTex", data.specularTexture);
                data.compositeMaterial.SetTexture("_AmbientTex", data.ambientTexture);
                Blitter.BlitTexture(context.cmd, data.sceneTexture, new Vector4(1, 1, 0, 0), data.compositeMaterial, 0);
            });
        }

        // 5) Final pass: Blit compositeOutputTexture into resourceData.activeColorTexture
        // We cannot sample activeColorTexture and write back into it in the same composite pass.
        // Therefore, we write the composite pass to compositeOutputTexture in 4) first,
        // then use this pass to copy that result back into activeColorTexture.
        using (var builder = renderGraph.AddRasterRenderPass<FinalBlitPassData>("SSSS Final Blit Pass", out var passData))
        {
            passData.sourceTexture = compositeOutputTexture;
            passData.blitMaterial = compositeMaterial;

            builder.UseTexture(passData.sourceTexture);
            builder.SetRenderAttachment(resourceData.activeColorTexture, 0);
            builder.AllowPassCulling(false);

            // Material pass 1 handles the copying.
            builder.SetRenderFunc(static (FinalBlitPassData data, RasterGraphContext context) =>
            {
                Blitter.BlitTexture(
                    context.cmd,
                    data.sourceTexture,
                    new Vector4(1, 1, 0, 0),
                    data.blitMaterial,
                    1
                );
            });
        }
    }

    public void Dispose()
    {
        // Nothing manual to release here. RenderGraph owns the temporary textures.
    }
}
```
</details>

*Show diffuse, ambient and specular parts*
---
## 2. Blur the diffuse lighting

With the RenderGraph pipeline setup now, you may have also noticed that in `SSSSRenderPass.cs`, we had references to several materials in passes 3, 4 and 5. These are intentional. They are custom shader materials used to **perform the actual image processing work** in those passes. Here, we will write our custom shader for our separable subsurface scattering using Jorge Jimenez's artist friendly kernel. 

Jorge Jimenez's paper on [**separable subsurface scattering**](https://www.cg.tuwien.ac.at/research/publications/2015/Jimenez_SSS_2015/Jimenez_SSS_2015-paper.pdf#page=0.99) shows us how subsurface scattering can be approximated not only more efficiently but also more physically accurate in screen space. 

At a high level, an expensive approach would be a full 2D convolution with a diffusion profile. However, Jimenez et al. show that this can be sped up using two cheaper 1D passes: one along the horizontal axis and one along the vertical axis. This works under useful assumptions about how irradiance behaves across the surface. This result is introduced through the **pre-integrated kernel**.

For our actual shader, I use the paper's **artist-friendly model**. It follows the same separable two-pass structure, but trades off some physical accuracy for more intuitive controls so that we can tweak the aesthetics of the results better. 

#### 2.1 The kernel

Our 1D kernel is a weighted blend of a near-range and far-range Gaussian:

$$
G(r,\sigma)
=
\frac{1}{\sqrt{2\pi}\sigma}
e^{-\frac{r^2}{2\sigma^2}}
$$

The artist-friendly kernel is then written as:

$$
a_m(r) = wG(r,\sigma_\text{near}) + (1-w)G(r,\sigma_\text{far})
$$

Here, $$w$$ controls the balance between short-range and long-range scattering, while $$\sigma_\text{near}$$ and $$\sigma_\text{far}$$ control the width of the near and far Gaussians respectively.

The full separable 2D kernel is simply:

$$
A_m(x,y) = a_m(x)a_m(y)
$$

which is represented implicitly in the code later by applying the same 1D convolution twice.

#### 2.2 The 1D Convolutions

Before we can write the code, we must firstly understand how the convolution works. 

Again at a high level, convolution means that for each pixel, we look at nearby samples, give each sample a weight, and blend them together. Samples close to the centre usually contribute more, while samples further away contribute less. For our SSS blur, those weights come from the artist-friendly kernel described above.

We can describe the continuous 1D convolution as:

$$
C_\text{out}(u)
=
\int_{-\infty}^{\infty}
C(u')a_m(u-u')\,du'
$$

In code, however, we cannot sample infinitely many points. So we approximate the integral with a finite weighted sum:

$$
C_\text{out}(u)
=
\frac{
\sum_{i=-N}^{N}
C(u + i\Delta u)a_m(|i|)
}{
\sum_{i=-N}^{N}
a_m(|i|)
}
$$

Here, $$N$$ is the number of steps on each side of the current pixel, $$\Delta u$$ is the spacing between samples, and $$a_m(\lvert i \rvert)$$ is the kernel weight for the $$i$$-th sample.

We use $$\lvert i \rvert$$ because the kernel only depends on distance from the centre, not whether the sample is to the left, right, above, or below the current pixel. In our case, we only sample along one axis at a time. For the horizontal pass, the samples move left and right. For the vertical pass, the samples move up and down.

The division by the total kernel weight normalises the result, keeping the brightness stable when we adjust the kernel parameters.

To avoid rewriting the same logic for the horizontal and vertical passes, we only change the sampling direction. Conceptually, the offset for each sample is:

$$
\Delta uv
=
\text{direction} \cdot \text{texelSize} \cdot \_BlurScale
$$

So the $$i$$-th sample position becomes:

$$
uv_i
=
uv + i\Delta uv
$$

A direction of $$(1,0)$$ gives us the horizontal pass, while a direction of $$(0,1)$$ gives us the vertical pass.

With that in mind, the shader code becomes a direct translation of the concepts above, but evaluated separately for each colour channel. This matters because red, green, and blue light can scatter differently within the same material.

<details class="collapsible" markdown="1">
<summary>
  <span class="collapsible-label">Show:</span>
  <span class="collapsible-meta"><code>ArtistFriendlyKernel.shader</code></span>
</summary>

```hlsl
Shader "bentoBAUX/FX/ArtistFriendlyKernel"
{
    Properties
    {
        _ScatterWeights("Scatter Weights (RGB)", Vector) = (1., 0.1, 0.1)
        _NearFarBalance("Near/Far Balance", Range(0,1)) = 0.5
        _SigmaNear("Near Sigma", Vector) = (3.5,1.5,0.8)
        _SigmaFar("Far Sigma", Vector) = (6.0, 3.0, 1.2)
        _BlurScale("Blur Scale", Float) = 1.0
        _StepCount("Step Count", Float) = 6
    }

    HLSLINCLUDE
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
    #include "Packages/com.unity.render-pipelines.core/Runtime/Utilities/Blit.hlsl"

    float3 _ScatterWeights;
    float _NearFarBalance;
    float3 _SigmaNear;
    float3 _SigmaFar;
    float _BlurScale;
    float _StepCount;

    SAMPLER(sampler_BlitTexture);

    float Gaussian1D(float r, float sigma)
    {
        const float INV_SQRT_2PI = 0.3989422804;
        return INV_SQRT_2PI / sigma * exp(-(r * r) / (2.0 * sigma * sigma));
    }

    float4 Convolve(float2 uv, float2 direction)
    {
        float2 texelSize = 1.0 / _ScreenParams.xy;

        float3 sum = 0.0;
        float totalweight = 0.;

        for (int i = -_StepCount; i <= _StepCount; i++)
        {
            float offset = abs((float)i);
            float2 sampleUV = clamp(uv + direction * texelSize * i * _BlurScale, 0., 1.);
            float3 sampleColour = SAMPLE_TEXTURE2D(_BlitTexture, sampler_BlitTexture, sampleUV).rgb;

            // Calculate weight w_i
            float3 w_i;
            w_i.r = max(_ScatterWeights.r,0.01) * (_NearFarBalance * Gaussian1D(offset, max(_SigmaNear.r, 0.01)) + (1 - _NearFarBalance) * Gaussian1D(offset, max(_SigmaFar.r, 0.01)));
            w_i.g = max(_ScatterWeights.g,0.01) * (_NearFarBalance * Gaussian1D(offset, max(_SigmaNear.g, 0.01)) + (1 - _NearFarBalance) * Gaussian1D(offset, max(_SigmaFar.g, 0.01)));
            w_i.b = max(_ScatterWeights.b,0.01) * (_NearFarBalance * Gaussian1D(offset, max(_SigmaNear.b, 0.01)) + (1 - _NearFarBalance) * Gaussian1D(offset, max(_SigmaFar.b, 0.01)));

            sum += sampleColour * w_i;
            totalweight += w_i;
        }

        return float4(sum/max(totalweight,0.0001), 1.0);
    }
    
    float4 HorizontalBlur(Varyings input) : SV_Target
    {
        return Convolve(input.texcoord, float2(1,0));
    }

    float4 VerticalBlur(Varyings input) : SV_Target
    {
        return Convolve(input.texcoord, float2(0,1));
    }

    ENDHLSL

    SubShader
    {
        Tags
        {
            "RenderPipeline" = "UniversalPipeline"
        }
        ZWrite Off
        ZTest Always
        Cull Off

        // Pass 0 performs the horizontal blur
        Pass
        {
            Name "Horizontal Blur"
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment HorizontalBlur
            ENDHLSL
        }

        // Pass 1 performs the vertical blur
        Pass
        {
            Name "Vertical Blur"
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment VerticalBlur
            ENDHLSL
        }
    }
}
```
</details>

In Unity, this shader is used through a material. We assign that material in the render feature settings, and the RenderGraph pass later uses it for the horizontal and vertical SSS blur passes in sections 3a and 3b in `SSSSRenderPass.cs`.

*Show intermediate result*

---
## 3. Combine everything

Now that we have finished adding subsurface scattering to our diffuse lighting, we need to put everything back together. According to our setup in section 4 of `SSSSRenderPass.cs`, we should now have five textures:

- `_SceneTex`: The current camera colour texture before the final SSS composite. This is used as a based image so that non-SSS objects remain untouched.
- `_DiffuseTex`: The unprocessed diffuse lighting of SSS objects.
- `_ProcessedDiffuseTex`: The diffuse lighting after two SSS blur passes.
- `_SpecularTex`: The specular lighting of SSS objects.  
- `_AmbientTex`: The ambient or indirect lighting of the SSS objects. This and `_SpecularTex` were kept separate so they do not get blurred.

Putting these together is simply computing their sum. However, because our diffuse texture was originally rendered to `SV_Target0` which is the colour output that ends up in the scene colour, `_SceneTex` already includes the diffuse texture. As a result, `_DiffuseTex` does not need to be included in the sum.

Another caveat to consider is **masking**. Since `_ProcessedDiffuseTex` is created by blurring the diffuse lighting in screen space, the blur does not automatically know where the SSS object ends. Without a mask, the processed diffuse contribution can bleed past the surface silhouette and appear as a glowing outline around the model.

*Include a picture of no mask*

To prevent this, we create a simple mask from `_DiffuseTex`. The idea is straightforward: pixels belonging to the SSS object have diffuse lighting, while unrelated scene pixels are usually black in this buffer.

First, we take the brightest colour channel of the diffuse buffer:

$$
m_\text{source} = \max(D_r, D_g, D_b)
$$

Then we convert this value into a soft mask using `smoothstep`:

$$
m = \operatorname{smoothstep}(a, b, m_\text{source})
$$

Here, $$a$$ is the lower threshold and $$b$$ is the upper threshold of the mask transition. Values below $$a$$ are treated as outside the SSS object and become $$0$$. Values above $$b$$ are treated as part of the SSS object and become $$1$$. Values between $$a$$ and $$b$$ are smoothly interpolated, giving us a soft boundary instead of a harsh binary edge.

In my case, I found that $$a = 0.01$$ and $$b = 0.1$$ worked well. This means that very dark diffuse values are ignored, while pixels with a stronger diffuse contribution are fully included in the SSS mask.

With this mask, we can now recombine everything:

$$
C_\text{final} = C_\text{scene} + C_\text{ambient} + C_\text{specular} + m \cdot C_\text{processed}
$$

> Do we subtract diffuse from the scene texture first?
> {: .title}
> One might think that the correct approach is to subtract `_DiffuseTex` from `_SceneTex` before adding `_ProcessedDiffuseTex`. However, in this implementation, that removes too much of the original high-frequency diffuse detail, such as local shading, texture variation, and small surface features. Since `_ProcessedDiffuseTex` is heavily blurred, using it as a full replacement makes the material look waxy and unnatural.
>
> Instead, I keep the original scene colour and overlay the processed diffuse contribution on top. This is not a perfectly physical composition, but it gives a more stable and visually convincing result for this screen-space implementation.
{: .box-info}

There is one last RenderGraph detail. We cannot safely read from the `resourceData.activeColorTexture` and write back into that same texture in the same pass. The composite pass therefore writes into a temporary output texture first. After that, we run a small copy pass that copies this temporary result back into the active scene colour texture.

Below is the final code for the compositor:

<details class="collapsible" markdown="1">
<summary>
  <span class="collapsible-label">Show:</span>
  <span class="collapsible-meta"><code>Compositor.shader</code></span>
</summary>

```hlsl
Shader "bentoBAUX/Util/Compositor"
{
    HLSLINCLUDE
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
    #include "Packages/com.unity.render-pipelines.core/Runtime/Utilities/Blit.hlsl"

    TEXTURE2D(_SceneTex);
    SAMPLER(sampler_SceneTex);

    TEXTURE2D(_DiffuseTex);
    SAMPLER(sampler_DiffuseTex);

    TEXTURE2D(_ProcessedDiffuseTex);
    SAMPLER(sampler_ProcessedDiffuseTex);

    TEXTURE2D(_SpecularTex);
    SAMPLER(sampler_SpecularTex);

    TEXTURE2D(_AmbientTex);
    SAMPLER(sampler_AmbientTex);

    float4 CompositeFrag(Varyings input) : SV_Target
    {
        float2 uv = input.texcoord;

        float3 sceneColour = SAMPLE_TEXTURE2D(_SceneTex, sampler_SceneTex, uv).rgb;
        float3 diffuse = SAMPLE_TEXTURE2D(_DiffuseTex, sampler_DiffuseTex, uv).rgb;
        float3 processed = SAMPLE_TEXTURE2D(_ProcessedDiffuseTex, sampler_ProcessedDiffuseTex, uv).rgb;
        float3 specular = SAMPLE_TEXTURE2D(_SpecularTex, sampler_SpecularTex, uv).rgb;
        float3 ambient = SAMPLE_TEXTURE2D(_AmbientTex, sampler_AmbientTex, uv).rgb;

        // Mask out the irrelevant scene objects
        float maskSource = max(diffuse.r, max(diffuse.g, diffuse.b));
        float mask = smoothstep(0.01, 0.1, maskSource);
        float3 maskTex = (float3)mask; // Debugging purposes

        float3 finalColour = sceneColour + ambient + specular + mask * processed;

        return float4(finalColour, 1.0);
    }

    float4 CopyFrag(Varyings input) : SV_Target
    {
        return SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearClamp, input.texcoord);
    }
    ENDHLSL

    SubShader
    {
        Tags
        {
            "RenderPipeline"="UniversalPipeline"
        }

        ZWrite Off
        ZTest Always
        Cull Off

        // Material pass 0 is our compositor.
        Pass
        {
            Name "Composite"

            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment CompositeFrag
            ENDHLSL
        }

        // Material pass 1 copies the output from the compositor into the active screen texture.
        Pass
        {
            Name "Copy"

            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment CopyFrag
            ENDHLSL
        }
    }
}
```
</details>

## Final result and backlighting limitation

At this point, the screen-space SSS pipeline is working: we split the lighting, blur the diffuse component, and composite everything back into the scene.

However, this only handles light scattering across the visible surface. It **does not automatically create the strong translucent backlighting effect** you often see around ears, fingers, or thin skin regions. That kind of effect depends on light travelling through the object, which our screen-space blur does not know about.

To approximate this, I added a simple backlighting term directly in `SSSS Master.shader`, based on Jorge Jimenez’s translucency approximation from his [**website**](https://www.iryoku.com/translucency/?utm_source=openai). The output of `CalculateTransmittance()` should be added as an extra term in the master shader. In my implementation, I add it together with the ambient contribution that will be written to `_AmbientTex`.

<details class="collapsible" markdown="1">
<summary>
  <span class="collapsible-label">Show:</span>
  <span class="collapsible-meta"><code>SSSSTransmission.shader</code></span>
</summary>

```hlsl
#ifndef SSSSTRANSMISSION_INCLUDED
#define SSSSTRANSMISSION_INCLUDED

// Code from: https://www.iryoku.com/translucency/?utm_source=openai
float3 T(float s)
{
    return float3(0.233, 0.455, 0.649) * exp(-s * s / 0.0064) +
        float3(0.1, 0.336, 0.344) * exp(-s * s / 0.0484) +
        float3(0.118, 0.198, 0.0) * exp(-s * s / 0.187) +
        float3(0.113, 0.007, 0.007) * exp(-s * s / 0.567) +
        float3(0.358, 0.004, 0.0) * exp(-s * s / 1.99) +
        float3(0.078, 0.0, 0.0) * exp(-s * s / 7.41);
}

float3 CalculateTransmittance(Surf surfaceData, Light lightData)
{
    float3 N = normalize(surfaceData.normalWS);
    float3 L = normalize(lightData.direction);

    float irradiance = max(0.3 + dot(-N, L), 0.0);
    float lightAtten = lightData.distanceAttenuation * lightData.shadowAttenuation;

    float s = surfaceData.thickness;

    float3 transmittance = T(s) * lightData.color * lightAtten * surfaceData.baseColor.rgb * irradiance;

    return transmittance;
}
#endif

```
</details>
