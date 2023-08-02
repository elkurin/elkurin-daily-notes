# Hardware Overlay and Overlay Processor

Hardware overlay is a technique in graphics to improve the display.  
It assignes a dedicated memory buffer to each divided surface and directly composits multiple buffers to the screen with overlaying each other.

![](https://hackmd.io/_uploads/BymGFxs82.png)


In this example, the video frame is underlaied the youtube window frame which contains transient area in the middle of the window where it shows video.  

In this way, you don't have to go through GPU frequently, and no need to produce a whole frame but only where it's necessary.  
In Chrome, there is a flag called [#overlay-strategies](chrome://flags/#overlay-strategies) to switch HW overlay strategies.  

## How Overlay is implemented
On drawing frame by [DirectRenderer::DrawFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/direct_renderer.cc;l=218;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3), it interacts with [viz::OverlayProcessorInterface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_interface.h).
[viz::OverlayProcessorInterface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_interface.h) is responsible for seperating the contents that should be sent into the overlay system and the contents that requires compositing from [viz::DirectRenderer](https://source.chromium.org/chromium/chromium/src/+/main:components/viz/service/display/direct_renderer.h), a renderer which does not delegate rendering to another compositor.

After drawing a frame, it calls [ProcessForOverlays](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/direct_renderer.cc;l=317;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3) and attempts to replace the quads of the root render pass with overlays. Its impelementation in Lacros is [OverlayProcessUsingStrategy::ProcessForOverlays](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_using_strategy.cc;l=318;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3).
Inside ProcessForOverlays, it attemps to overlay with each strategies by calling [AttempWithStrategies](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_using_strategy.cc;l=729;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3).
Here, [`strategies_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_using_strategy.h;l=107;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3) represent how they overlay that are implemented in classes inheritting [OverlayProcessorStrategy](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_strategy.h;l=23;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3). Strategies are listed below:
- [OverlayStrategyFullscreen](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_strategy.h;l=23;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3): Promote a single full screen quad. Buffer covers whole screen, so we can scan out the contents directly.
- [OverlayStrategySingleOnTop](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_strategy_single_on_top.h): Single element from a different source not occluded.
- [OverlayStrategyUnderlay](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_strategy_underlay.h): Single element with some UI on top. Looks for a video quad without regard to quads above it.

In [AttempWithStrategies](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_using_strategy.cc;l=729;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3), first it tries to [Propose](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_using_strategy.cc;l=744;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3) all strategies and generates [`proposed_candidates`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_using_strategy.cc;l=742;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3). Then [Attempt](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_using_strategy.cc;l=782;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3) for appropriate strategies and flip `used_overlay` flag to true if it actually used overlay. If `used_overlay` is set to true, [AdjustOutputSurfaceOverlay](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_using_strategy.cc;l=829;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3).
These Propose, Attemp, AdjustOutputSurfaceOverlay are implemented for each strategies.

## What OverlayStrategy does
[OverlayStrategySingleOnTop](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_strategy_single_on_top.cc) is the simplest strategy.  
On [Propose](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_strategy_single_on_top.cc;l=89;drc=ad4f0cc76bca3d9c79b887b061c9a5bbea287585)-ing this strategy, it generates [OverlayCandidateFactory](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_candidate_factory.cc;l=174;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3) from render pass.

It handles each DrawQuad stored in [`quad_list`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/quads/render_pass_internal.h;l=90-91;drc=8ba1bad80dc22235693a0dd41fe55c0fd2dbdabd) which is stored in front-to-back order.  
[FromDrawQuad](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_candidate_factory.cc;l=115;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3) takes each DrawQuad in [`quad_list`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/quads/render_pass_internal.h;l=90-91;drc=8ba1bad80dc22235693a0dd41fe55c0fd2dbdabd) and proceed overlay by calling From*Quad function corresponds to its type.  
For example, [FromTextureQuad](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_candidate_factory.cc;l=532;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3) is called for texture quad, and it clips and transforms the quad with the data in DrawQuad inside [FromDrawQuadResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_candidate_factory.cc;l=299;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3).  
[`display_rect`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_candidate_factory.cc;l=328;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3) represents a rectangular area to clip.

### What is RenderPass
[RenderPass](https://vulkan-tutorial.com/Drawing_a_triangle/Graphics_pipeline_basics/Render_passes) is a rendering pipeline. RenderPass renders an output image into a set of frame buffer and it represents chain of tasks and dependency chain of sub frames required.  
In overlay, there are several frames overlaying each other with the specific order. There may be an opration to transform a frame to another size, or clip the window by some smaller shape. IIUC, producing such frames and modifying how it's displayed are the cahin of tasks and dependency.  
TODO(elkurin): Understand deeper.

In Chromium, [RenderPassInternal](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/quads/render_pass_internal.h) represents the data shared between the compositor and aggregated render passes.  
[`quad_list`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/quads/render_pass_internal.h;l=90-91;drc=8ba1bad80dc22235693a0dd41fe55c0fd2dbdabd) stores [DrawQuad](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/quads/draw_quad.h;l=35;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3) data in front-to-back order.  
DrawQuad is added to/emitted from `quad_list` from [viz::SurfaceAggregator](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/surface_aggregator.h;drc=c66714e08ecc55bb827e8db90339e1b8903d5ce3)

[viz::AggregatedRenderPass](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/quads/aggregated_render_pass.h) represents a render pass that is a result of aggregating render passes from all of the relevant surfaces.

### What is Quad
Quad is a terminology used in chromium which represents a unit of rectangular area to draw. It's called "quad" since each area is rectangular.  
[viz::DrawQuad](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/quads/draw_quad.h;l=35;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3) holds a bag of data used for drawing a quad.

### Many rect
There are many *_rect params in [OverlayCandidate](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_candidate.h).
- `display_rect`: rect in the content space. is the bounds when combined with `transform`
- `uv_rect`: crop within the buffer to be placed inside `display_rect`
- `clip_rect`: rect to clip after composition

Overlay should handle these rect on Client side, but it currently does not (may not) for `clip_rest`. It does only when [`can_delegate_clipping`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_candidate_factory.cc;l=398;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3) is true.  
If not, it directly aplies clipping to `display_rect` and `uv_rect` by [DoGeometricClipping](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_candidate_factory.cc;l=407;drc=73265a30d5b5a3873dbf3b92f01abc5c2892ece3).
