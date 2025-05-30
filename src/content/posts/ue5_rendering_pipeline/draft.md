---
title: UE5/Rendering Pipeline (Draft)
published: 2025-05-16
description: ''
image: ''
tags: [rendering, graphics, ue5, pipeline]
category: 'UE5'
draft: true 
lang: 'zh'
---

# Introduction

# PC/Console Rendering Pipeline

## Deferred Shading

* Update sky atmosphere
* Update lumen scene
* Nanite begin visibility query
* Prepare distance field scene
* Shading energy conservation
* Glint shading luts
* Shadow scene renderer begin render
* Initailzie raytracing flags
* Raytracing begine gather instances
* SVT begin async update
* Nanite streaming begin async update
* Subsurface profile update texture
* Specular profile update texture
* Rect light atlas update texture
* IES atlas update texture
* Substrace preinit views
* Scene textures initialize
* Begin init views
* IPersistentViewUniformBufferExtensions prepare view
* Prepare raytracing decals
* Prepare pathtracing clould material
* Get raytracing lighting miss shader
* Gather light function lights 
* Finish gather dynamic mesh elements
* FX system pre render
* GPUScene upload dynamic primitve shader data
* Begin deferred culling
* Update physics field
* Update scene uniform buffer
* Virtual Texture System endupdate
* Raytracing finish gather instances
* Launch gather and sort lights tasks
* Scene extentions renderers PreRender
* Dithered stencil fill pass
* Nanite streaming end async update
* Nanite streaming request updates
* End init views
* Substrace initialize frame scene data
* Hair strands bookmark
* Render sky atmosphere lookup table
* Render water info texture
* Creat dbuffer textures
* Custom render pass
* Render prepass and velocity
* Nanite build shading commands
* VRS prepare image based
* Finish init dynamic shadows
* Hair strands bookmark
* Render occlusion
* ViewExtension pre render base pass
* Gather lights and compute light grid
* Render light function atlas
* Init volumetric clouds
* Render sky atmosphere lookup table
* Begin async distance field shadow projection
* Init local fog volumes
* Init volumetric render target
* Render sky atmosphere lookup table
* Capture sky env map
* Render custom depth pass
* Update lumen scene
* Render shadow depth maps
* Composition lighting process before base pass
* Dispatch ray tracing world updates
* Setup ray tracing light data
* Begin gather lumen surface cache feedback
* Render lumen scene lighting
* Render Base Pass
* Resolve scene depth pass
* Setup exposure illuminance pass
* Extract normals for next frame reprojection
* Validate sky light realtime capture
* Render occlusion
* Hair strands pre pass
* Hair strands base pass
* Render heterogeneous volumen shadows
* Substrace material classification pass
* Substrace dbuffer pass
* Copy stencil to lighting channel texture
* Render single layer water depth prepass
* Dispatch async lumen indirect lighting work
* Render volumetric cloud
* Render shadow depth maps
* Nanite streaming submit frame streaming requests
* Render custom depth
* Render velocities
* Resolve scene depth
* Composition lighting process after base pass
* Render diffuse indirect and ambient occlusion
* Render indirect capsule shadows
* Render DFAO indirect shadow
* Render Lights
* Render Mega Lights
* Store stochastic lighting scene history
* Inject translucent lighting volume ambient cubemap
* Filter translucent lighting volume
* Render diffuse indirect and ambient occlusion
* Render deferred reflections and sky lighting
* Subsurface pass
* Substrace opaque rough refraction pass
* Hair strands scene color scattering
* Render ray tracing sky light
* Composite ray tracing sky light
* Compute volumetric fog
* Render heterogeneous volume
* Render volumetric cloud
* TSR measure flickering luma
* Render light shaft occlusion
* Render sky atmosphere
* Render Fog
* Render local fog volume
* Render volumetric cloud
* Compose volumetric render target over scene
* Render volumetric cloud
* Render translucent under water
* Render single layer water
* Calculate exposure illuminance
* Render Opaque FX
* Hair composition
* Nanite draw visible bricks
* Compite heterogeneous volume
* Render ray tracing translucent
* Render front layer translucency
* Render lumen front layer translucency reflections
* OIT add sort triangles
* Render translucency
* Hair composition
* Render distortion
* Render velocities
* Render light shaft bloom
* Upscale translucency
* Render pathtracing
* Virtual texture feedback end
* Render physics field
* Render distance field lighting
* Render mesh distance field visualization
* Postprocessing
* Finish gather lumen surface cache feedback