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
  <span class="collapsible-meta"><a href=""><code>SSSSRenderFeature.cs</code></a></span>
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
<details class="collapsible" markdown="1">
<summary>
  <span class="collapsible-label">Show:</span>
  <span class="collapsible-meta"><a href=""><code>SSSSRenderPass.cs</code></a></span>
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
        TextureHandle compositeOutputTexture = UniversalRenderer.CreateRenderGraphTexture(renderGraph,desc,"_SSSSCompositeOutputTexture",false);

        // Make them available to later passes in this frame.
        SSSSFrameData customData = frameData.Create<SSSSFrameData>();
        customData.diffuseTexture = diffuseTexture;
        customData.processedDiffuseTexture = processedDiffuseTexture;
        customData.specularTexture = specularTexture;
        customData.ambientTexture = ambientTexture;

        // 2) Raster pass: attach diffuse and specular into separate render textures for SSS processing
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

            builder.SetRenderFunc(static (MRTPassData data, RasterGraphContext context) =>
            {
                context.cmd.ClearRenderTarget(false, true, Color.black);
                context.cmd.DrawRendererList(data.rendererList);
            });
        }

        // 3a) SSS horizontal pass: Apply horizontal SSS pass onto diffuse texture and output it to horizontalTempTexture.
        using (var builder = renderGraph.AddRasterRenderPass<SSSSPassData>("SSSS SSS Horizontal Pass", out var passData))
        {
            passData.inputTexture = diffuseTexture;
            passData.material = SSSSMaterial;

            builder.UseTexture(passData.inputTexture);
            builder.SetRenderAttachment(horizontalTempTexture, 0);
            builder.AllowPassCulling(false);

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
            SSSSFrameData custom = frameData.Get<SSSSFrameData>();

            passData.sceneTexture = resourceData.activeColorTexture;
            passData.diffuseTexture = custom.diffuseTexture;
            passData.processedDiffuseTexture = custom.processedDiffuseTexture;
            passData.specularTexture = custom.specularTexture;
            passData.ambientTexture = custom.ambientTexture;
            passData.compositeMaterial = compositeMaterial;

            builder.UseTexture(passData.sceneTexture);
            builder.UseTexture(passData.diffuseTexture);
            builder.UseTexture(passData.processedDiffuseTexture);
            builder.UseTexture(passData.specularTexture);
            builder.UseTexture(passData.ambientTexture);

            builder.SetRenderAttachment(compositeOutputTexture, 0);
            builder.AllowPassCulling(false);

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
        using (var builder = renderGraph.AddRasterRenderPass<FinalBlitPassData>("SSSS Final Blit Pass", out var passData))
        {
            passData.sourceTexture = compositeOutputTexture;
            passData.blitMaterial = compositeMaterial;

            builder.UseTexture(passData.sourceTexture);
            builder.SetRenderAttachment(resourceData.activeColorTexture, 0);
            builder.AllowPassCulling(false);

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

---
## 2. Blur the diffuse lighting
---
## 3. Combine everything
