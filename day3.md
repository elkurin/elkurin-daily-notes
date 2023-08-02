# ChromeOS compositing - How Frame is Sent

might be not accurate... 許してね

## Compositing Window Manager Model (on ChromeOS)
Client produces a frame, updates the frame buffer and pass its ownership to server when drawing is done.  
Server receives the ownership of multiple frames, composites them to produce whole screen frame and releases their ownership to give them back to the client.  
When the ownership of the frame buffer is released, Client swaps the frame (if exists) prodocued while Server was compositing and do the same thing above.  

This frame buffer is attached on initialization and must be resized when window states or bounds are changed.  

On ChromeOS, Server is ChromeOS itself and Client can be applications or Lacros (ChromeOS-specific browser).  

*Note: Compositor does not simply use the given buffer. Since this model uses IPC, the states may be not synced between Server and Client. There is a mechanism to ensure that Server and Client agrees on the states and then apply the state to the frame.*

## ChromeOS window manager protocol
ChromeOS uses wayland protocol: https://wayland-book.com/.  
Wayland is a protocol between display server and client mainly used on Linux and other Unix OS. (ChromeOS is also based on Linux)  

In ChromeOS display server is in Ash proces.  
To support ChromeOS fully, we expands wayland protocol.  
Codes: [protocols](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/), [server side impl](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/), [client side impl](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/;drc=7b64008d33791c40cbcc11517900633c106cdce9).

## How frame is sent
When to send/receive frames is controlled and triggered by compositor on Server.  

When compositor is ready to receive new frames, Server sends FrameCallback.
This triggers `wl_callback_send_done` and Client is notified that compositor is ready.  
Then Client looks up [`pending_frames_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.h;l=211;drc=131eb03f815825c6641ddafb06335178c8a2d237) and commits the appropriate frame. It sends `wl_surface_frame` message to Server which triggers Server to request the next frame callback.  
Continues this loop...


### In Code
#### Server side
[SurfaceTreeHost::DidReceiveCompositorFrameAck()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface_tree_host.cc;l=176-182;drc=131eb03f815825c6641ddafb06335178c8a2d237) runs frame callback. This is called when CompositorFrame given to SubmitCompositorFrame() has been processed and that another frame can be submitted ([ref](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_frame_sink_client.h;l=53-57;drc=131eb03f815825c6641ddafb06335178c8a2d237)).  

More details:
[RequestFrameCallback](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.cc;l=397;drc=131eb03f815825c6641ddafb06335178c8a2d237) sets frame callback to [`pending_state_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.h;l=639;drc=131eb03f815825c6641ddafb06335178c8a2d237) and `pending_state_.frame_callbacks` is [passed to `cashed_state_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.cc;l=875-876;drc=131eb03f815825c6641ddafb06335178c8a2d237) on [Surface::Commit()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.cc;l=846;drc=131eb03f815825c6641ddafb06335178c8a2d237), and `cached_state_` is passed to `state_` in [Surface::CommitSurfaceHierarchy()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.cc;l=917;drc=131eb03f815825c6641ddafb06335178c8a2d237). It's appended to surface hierarchy callbacks by [SurfaceTreeHost::SubmitCompositorFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface_tree_host.cc;l=285;drc=131eb03f815825c6641ddafb06335178c8a2d237).   
Then it goes around viz or renderer to produce a frame and comes back to DidReceiveCompositorFrameAck when the previous CompositorFrame has been processed and another frame can be submitted.  

#### Client side
`wl_callback_send_done`'s implementation on Client is [WaylandFrameManager::FrameCallbackDone()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=445;drc=131eb03f815825c6641ddafb06335178c8a2d237). Inside FrameCallbackDone(), Client start processing pending frames ([MaybeProcessPendingFrame()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=138;drc=131eb03f815825c6641ddafb06335178c8a2d237)). `pending_frames_` consists of the frame produced on viz and waiting to be committed.  
On fiding a proper frame, it calls [PlayBackFrame()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=223;drc=131eb03f815825c6641ddafb06335178c8a2d237) from [here](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=204;drc=131eb03f815825c6641ddafb06335178c8a2d237).  
On PlayBackFrame(), it first [ApplySurfaceConfigure](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=281;drc=131eb03f815825c6641ddafb06335178c8a2d237) and then [Commit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=290;drc=131eb03f815825c6641ddafb06335178c8a2d237).  
[WaylandFrameManager::ApplySurfaceConfigure](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=321;drc=4dd3677c79a7d76f524f17c9994dc11ab09de172) sends `wl_surface_frame` message.  
[WaylandSurface::Commit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_surface.cc;l=303;drc=131eb03f815825c6641ddafb06335178c8a2d237) sends `wl_surface_commit` message.  

More details on how produced frames are stored in `pending_frames_`:
On PlayBackFrame(), it also calls [MaybeProcessSubmittedFrames](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=643;drc=4dd3677c79a7d76f524f17c9994dc11ab09de172) which triggers [OnSubmission](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_buffer_manager_host.cc;l=413;drc=4dd3677c79a7d76f524f17c9994dc11ab09de172). OnSubmission is sent to GPU process via mojo IPC called [OnSubmission](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/mojom/wayland_buffer_manager.mojom;l=122;drc=4dd3677c79a7d76f524f17c9994dc11ab09de172) and prompts GPU process to [MaybeSubmitFrames](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/mojom/wayland_buffer_manager.mojom;l=122;drc=4dd3677c79a7d76f524f17c9994dc11ab09de172).  
New frames are sent from GPU process via mojo IPC called [CommitOverlays](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/mojom/wayland_buffer_manager.mojom;l=84;drc=4dd3677c79a7d76f524f17c9994dc11ab09de172) and appended to `pending_frames_` by [RecordFrame()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=115;drc=131eb03f815825c6641ddafb06335178c8a2d237).  
Mojo IPC connects between [WaylandBufferManagerHost](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_buffer_manager_host.cc) in the browser process and [WaylndBufferManagerGPU](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/gpu/wayland_buffer_manager_gpu.cc) in the GPU process. 

#### Note about OnSubmission
OnSubmission is also triggered by compositor when compositor release a frame buffer.  
Compositor can explicitly release buffer by `zwp_linux_buffer_release_v1_send_fenced_release` message in [LinuxBufferRelease](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/zwp_linux_explicit_synchronization.cc) and Client calls [`release_callbacks_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_buffer_handle.h;l=88;drc=4dd3677c79a7d76f524f17c9994dc11ab09de172) on releasing buffer. `release_callbacks_` is set to [WaylandFrameManager::OnExplicitBufferRelease](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=588;drc=4dd3677c79a7d76f524f17c9994dc11ab09de172) which calls [MaybeProcessSubmittedFrames](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=620;drc=4dd3677c79a7d76f524f17c9994dc11ab09de172). From here, it's the same to the path explained the above section.

## How frame is produced
The above section explains how pending frames on Client are sent to Server.
We still need to understand how frames are produced.  
I'll leave that as my next task and finish this note here... 

