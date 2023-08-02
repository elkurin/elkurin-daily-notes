# Delegated Compositing

Delegated Compositing is a mechanism for compositing in Server-Client model to reduce the performance overhead.  
In short, Client delegates all compositing to Server.

## Naive Compositing
Client produces frames and Server composites the frames to entire display.

Sometimes, Client produces more than one frame and try to composit those frames on Client side and produce one frame representing entire Client's surface.  
This surface will be sent to Server and Server composits the frames with those received from another Client.  
In this flow, Client aggregates Quads and renders Quads to a single buffer and then Server uses this buffer as single Quad representing Client window to again composit whole display.

![](https://hackmd.io/_uploads/BJ_B5O2I3.png)

This requires GPU to composite twice.

## How Delegated Compositing Works
Instead, we forward each quad as is to Server and *delegate compositing* to Server at once.  
Each quad is a buffer passed to Server.

![](https://hackmd.io/_uploads/S1gi5d28n.png)

In this mehcanism, Client does not composite and we only composite once on Server side.  
Client forwards all quads in the root RenderPass.

In this mechanism, quads are observable on Server side, so there are a lot of ExoSurfaces under ExoShellSurfaceHost on Server.  
Each ExoSurface represents a sent quad.

## Implementation
### cc to viz
[cc::LayerTreeHostImpl::CalculateRenderPasses](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host_impl.cc;l=1270;drc=5992439d25f71ce29efa8db1c699b99e8773d41f) calculates RenderPass and [AppendQuads](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host_impl.cc;l=1391;drc=5992439d25f71ce29efa8db1c699b99e8773d41f) to the root render surface.  
When RenderPass becomes ready, it [DrawLayers](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host_impl.cc;l=2511;drc=5992439d25f71ce29efa8db1c699b99e8773d41f) and [SubmitCompositorFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host_impl.cc;l=2557;drc=5992439d25f71ce29efa8db1c699b99e8773d41f) whose implementation if [AsyncLayerTreeFrameSink](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/mojo_embedder/async_layer_tree_frame_sink.cc;l=147;drc=5992439d25f71ce29efa8db1c699b99e8773d41f) to bypass mojo API in [compositor_frame_sink.mojom](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:services/viz/public/mojom/compositing/compositor_frame_sink.mojom;l=67;drc=5992439d25f71ce29efa8db1c699b99e8773d41f).  
Mojo API passes RenderPass from ChromeCompositor(cc) to Viz.

### viz to browser process
In SubmitCompositorFrame flow, it eventually calls [Display::DrawAndSwap](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/display.cc;l=702;drc=5992439d25f71ce29efa8db1c699b99e8773d41f) as mentioned in [previous note](https://hackmd.io/@elkurin/S1x4KUeIn). Here, it [Aggregate](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/surface_aggregator.cc;l=2099;drc=5992439d25f71ce29efa8db1c699b99e8773d41f) frames as [AggregatedFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/aggregated_frame.h;l=23;drc=5992439d25f71ce29efa8db1c699b99e8773d41f) class which includes RenderPass list and SurfaceDamageRectList.  
Then these RenderPass will be forwarded to [ProcessForOverlays](https://source.chromium.org/chromium/chromium/src/+/main:components/viz/service/display/overlay_processor_interface.h;l=149;drc=f5f78068e1ae5ad6650bae00a812d344e58c6810). Its implementation for Delegated Compositing is in [OverlayProcessorDelegated](https://source.chromium.org/chromium/chromium/src/+/main:components/viz/service/display/overlay_processor_delegated.cc;l=268;drc=dd7f23595120c1385285eb015389d92d8f95dd6d). This is currently used by default on Lacros as you can see in [OverlayProcessorInterface::CreateOverlayProcessor](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/overlay_processor_interface.cc;l=141;drc=5992439d25f71ce29efa8db1c699b99e8773d41f).  
The results are sent via mojo IPC [CommitOverlays](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/mojom/wayland_buffer_manager.mojom;l=84;drc=c8b4f6bb5bf563e9e94f1069433cea1023177aba) and bypass tos [WaylandBufferManagerHost](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_buffer_manager_host.cc;l=270;drc=5992439d25f71ce29efa8db1c699b99e8773d41f) in the browser process.

### Client (browser process) to Server
Quads are sent via [CommitOverlays](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/gpu/gbm_surfaceless_wayland.cc;l=283-285;drc=5992439d25f71ce29efa8db1c699b99e8773d41f). A bag of data representing quads are passed as vector ot [WaylandOverlayConfig](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/common/wayland_overlay_config.h;l=21;drc=5992439d25f71ce29efa8db1c699b99e8773d41f).  
In Wayland, quad is [WaylandSubsurface](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_subsurface.h;l=24;drc=c1fb8bf3a129a61ac02f1718b76044faa00ec6ef). [WaylandSubsurface::ConfigureAndShowSurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_subsurface.cc;l=128;drc=5992439d25f71ce29efa8db1c699b99e8773d41f) forwards overlay by [`subsurface_place_above`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/wl_subcompositor.cc;l=32;drc=8ba1bad80dc22235693a0dd41fe55c0fd2dbdabd) or [`subsurface_place_below`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/wl_subcompositor.cc;l=39;drc=8ba1bad80dc22235693a0dd41fe55c0fd2dbdabd) implemented as wl_subsurface_interface. On Exo, Subsurface is also handled as ExoSurface. There is a class named [SubSurface](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/sub_surface.h), but it only provides functions for treating a surface as a sub-surface, not a surface itself.  
Quads are now ExoSurface and they are compositted as usual Surface on Server compositor.
