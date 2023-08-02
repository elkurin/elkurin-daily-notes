# ChromeOS compositing - How Frame is Produced

How Frame is sent: https://hackmd.io/@elkurin/SyzSuVlLh  
The previous note wrote about the connection between Client and Server.  
This note is about what Client does.  

might be not accurate 許してね

## Whole Picture
As explained in ["How Frame is sent"](https://hackmd.io/@elkurin/SyzSuVlLh) note, the browser process will send the pending frames to the compositor when the compositor requests Client to send.  

"Pending frames" are comming from GPU process.  
GPU produces and sends a frame once in the time interaval to the browser process which ends up being shown in the display.  

In Chromium, GPU triggers the event called "BeginFrame" to start producing/sending frame flow and its timing is controlled by VSync system.

## BeginFrame
BeginFrame is a terminology used in viz. BeginFrame is the message to start producing the frame and preset it.  

In this section, I go through the flow inside viz on ChromeOS.  
On BeginFrame, GPU process [AttempDrawAndSwap](https://source.chromium.org/chromium/chromium/src/+/main:components/viz/service/display/display_scheduler.cc;l=496;drc=0b14fa8ae1b7641810a827f1b475b103e6d8a1e3) in DisplayScheduler which will 1. Draw frame, and 2. Swap.  
After DrawAndSwap. it calls [GbmSurfacelessWayland::Present](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/gpu/gbm_surfaceless_wayland.cc;l=173;drc=1fbac4e1d9d8c232aba1f88276558464dc6d0e4d) which presents current frame asynchronously.  
The frame presented by the above function will be once stored to [`unsubmitted_frames_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/gpu/gbm_surfaceless_wayland.h;l=168;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267).
The stored frame will be submitted to browser process by [GbmSurfacelesWayland::MaybeSubmitFrames()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/gpu/gbm_surfaceless_wayland.cc;l=265;drc=4dd3677c79a7d76f524f17c9994dc11ab09de172) via mojo IPC called [CommitOverlays](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/mojom/wayland_buffer_manager.mojom;l=84;drc=4dd3677c79a7d76f524f17c9994dc11ab09de172).

### DrawFrame
[DirectRenderer::DrawFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/direct_renderer.cc;l=214;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267) draws a frame.  

First, it determines where to draw.  
To improve the performance, it skips drawing the area where the correct image is already drawn since it hasn't changed from the last frame. The actual area needed to be drawn is called "damaged area". This size is stored as [root_damage_rect](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/direct_renderer.h;l=136;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267).  

Inside [DrawRenderPassAndExecuteCopyRequests](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/direct_renderer.cc;l=584;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267), [DrawRenderPass](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/direct_renderer.cc;l=637;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267) phase draws and [CopyDrawnRederPass](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/skia_renderer.cc;l=3096;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267) copies the output of the current frame. TODO(elkurin): What is this copy doing?  

RenderPass contains QuadList in front-to-back order. Quad is a frame unit to draw whose shape is rectangle (so that it is called quad). DrawRenderPass will draw "Quad" in [DoDrawQuad](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/skia_renderer.cc;l=1208;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267), [DrawRenderPassQuad](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/skia_renderer.cc;l=3016;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267) and then [DrawSingleImage](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/skia_renderer.cc;l=2203;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267). Quad is a frame unit to draw whose shape is rectangle (so that it is called quad). [DrawQuad]  (https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/quads/draw_quad.h;l=35;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267) is a bag of data used for drawing.  

It [PrepareCanvas](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/skia_renderer.cc;l=1291;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267) and draws the image constructed in skia library [code](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/skia/src/core/SkCanvas.cpp;l=1974;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267).

### SwapBuffer
Frame buffer is swapped instead of recreated / copied or anything like that to save memory and speed performance.  

There are three classes inheritting DirectRenderer, [SkiaRenderer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/skia_renderer.h), [SoftwareRenderer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/software_renderer.h) and [NullRenderer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/null_renderer.h). TODO(elkurin): Figure out which does what.  

Here, we follow SkiaRenderer pass.  
[SkiaRenderer::SwapBuffers](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/skia_renderer.cc;l=943;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267) will swaps the buffer from the old one to the new one appropriately.  
Pending buffers to swap are handled by [BufferQueue](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display_embedder/buffer_queue.h;drc=f9f1937aad94e301a537e19dff88e837a91015df).  

As this is handled as a queue, there could be many pending buffers to be swapped. However it's usually required to draw and swap one by one without skipping.  
The maximum number of pending swap allowed to be skipped is determined by [MaxPendingSwaps](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/display_scheduler.cc;l=292;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267) and you can see it's set to 1 in [OutputSurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/output_surface.h;l=68;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267), but could be overridden by 2~4 on some platforms and in some condition such as [Android](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display_embedder/skia_output_device_buffer_queue.cc;l=196-203;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267).

### GPU Tasks

GPU tasks such as Rephase, SwapBuffer, CopyOutput [EnqueueGpuTask](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display_embedder/skia_output_surface_impl.cc;l=1329;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267) to [`gpu_tasks_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display_embedder/skia_output_surface_impl.h;l=419;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267).  
They are guaranteed to run in sequence.  

## VSync
GPU sends BeginFrame in the short interval constantly. The interval is decided as not to exceed the limit of the graphics capacity. This feature is called "VSync".   
"fps" or "frame rate" is how many time GPU sends frames in a second.  

[OnGPUVSync](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/frame_sinks/gpu_vsync_begin_frame_source.cc;l=25;drc=91ca46a556920a644e2215217b00c0fd32248308) is called every `vsync_interval` time which is set to [1/60 secs in viz](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/frame_sinks/begin_frame_args.h;l=160;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267) by default.  
OnGPUVSync calls [OnBeginFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/frame_sinks/begin_frame_source.cc;l=482;drc=10c56a7c2d1f9bf623a1bbdd506683a430594267) and starts BeginFrame flow.  

The above BeginFrame will be automatically triggered in the same time interval, produces a frame and send it to the browser process.  


## Additional Notes
vsync_interval is usually the same but this may change on throttling.  
vsync_timing can be manipulated from `zcr_vsync_timing_v1_send_update` message from [VsyncTimingManager](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/vsync_timing_manager.h).