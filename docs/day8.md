# Exo Surface Hierarchy

Each Client window looks like this on Server side:

```
ExoShellSurface-1<-1>
    ExoShellSurfaceHost-1<-1>
        ExoSurface-4<-1>
        ExoSurface-5<-1>
        ExoSurface-6<-1>
```

Who are they? What are they responsible for?

## ExoShellSurface
ExoShellSurface is constructed at [ShellSurfaceBase::CreateShellSurfaceWidget](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/shell_surface_base.cc;l=1459;drc=6eb246e1ed373c657324ec82ae79d9829e548df0).  
Creates [ShellSurfaceWidget](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/shell_surface_base.cc) stored in [`widget_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/shell_surface_base.h;l=437;drc=1246b0cf4ab728aceb9b833524fe181266f48634) and its window's name is set to [ExoShellSurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/shell_surface_base.cc).

ExoShellSurface is responsible for
1. Window state: Maximized, Minimized, Snapped...
2. Window property: Restore bounds, Raster scale...
3. Window bounds: the size of the window
4. Window activation: whether window is focused ([FocusManager](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/focus/focus_manager.h;l=118;drc=ac12f6366c92c4f41c06e2005598b1bc54b91c25))

Overall, it works as a container of states and properties of the window.

ExoShellSurface is a parent window of [ExoShellSurfaceHost](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/shell_surface_base.cc;l=1591;drc=6eb246e1ed373c657324ec82ae79d9829e548df0) AKA host window.

## ExoShellSurfaceHost
ExoShellSurfaceHost window is [`host_window_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface_tree_host.h;l=192;drc=ac12f6366c92c4f41c06e2005598b1bc54b91c25)  constructed in [ShellSurfaceBase](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/shell_surface_base.cc;l=306;drc=6eb246e1ed373c657324ec82ae79d9829e548df0) and stored in [SurfaceTreeHost](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/surface_tree_host.cc;l=103;drc=85a75213fb807442f7e48498cccd2cfe366992e4).　　
It contains surface tree hosted by `host_window_`.  

SurfaceTreeHost is responsible for managing CompositorFrame.  
[OnSurfaceCommit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/shell_surface_base.cc;l=963;drc=ac12f6366c92c4f41c06e2005598b1bc54b91c25), `host_window_` receives SurfaceID and submit CompositorFrame to [LayerTreeFrameSinkHolder](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/layer_tree_frame_sink_holder.h) which keeps the track of buffers.　　
It also stores [`frame_callbacks_`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/surface_tree_host.h;l=202;drc=cd40e412a651c6d477e76bb9e59a4022d1adf1f2) list to notify Client when to produce compositor frame.

## ExoSurface
[ExoSurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.h) represents a rectangulr area that is displayed on the screen. This size is not necessarily same to the bounds managed by ExoShellSurface.　　  
This is a unit of displayed area and frame buffer is attached to this surface to be visualized.

ExoSurface is responsible for showing the frame.

When buffer is attached　by [Surface::Attach](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.cc;l=365;drc=ac12f6366c92c4f41c06e2005598b1bc54b91c25), it is stored to [`pending_state_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.h;l=639;drc=ac12f6366c92c4f41c06e2005598b1bc54b91c25).　　  
On [Surface::Commit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.cc;l=846;drc=ac12f6366c92c4f41c06e2005598b1bc54b91c25), pending state is cached to [`cached_state_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.h;l=641;drc=ac12f6366c92c4f41c06e2005598b1bc54b91c25), and will be applied together with surface hierarchy by [CommitSurfaceHierrchy](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.cc;l=917;drc=ac12f6366c92c4f41c06e2005598b1bc54b91c25).

Commit message is sent from Client directly to ExoSurface.

## Summary
ExoShellSurface works as a container holding window state and other properties.　　  
ExoShellSurfaceHost hosts the surface tree and manages CompositorFrame.  
ExoSurface is a rectangle displayed area itself.