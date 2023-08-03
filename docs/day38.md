# Surface Augmenter

Delegate composition is a machenism that Client sends quads insted of a composited frame and lets Server compiste a frame the given quads.  
[`surface_augmenter`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml), [`augmented_surface`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=92;drc=a63b96fa534c4513465b194e5a39aa833d37205a) and [`sugmented_sub_surface`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=189;drc=a63b96fa534c4513465b194e5a39aa833d37205a) protocol are used to deliver delegated composition.

## Protocol
### [`surface_augmenter`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=27-90;drc=a63b96fa534c4513465b194e5a39aa833d37205a)

[`surface_augmenter`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=27-90;drc=a63b96fa534c4513465b194e5a39aa833d37205a) is an extended inteface which allows delegate composition of the surface contents.  
This effectively disconnects the direct relationship between the buffer and the surface content.  

- [`destroy`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=38;drc=a63b96fa534c4513465b194e5a39aa833d37205a): Informs Server that Client will no longer use this protocol object.
-[`create_solid_color_buffer`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=52;drc=a63b96fa534c4513465b194e5a39aa833d37205a): Instantiate a buffer of the given size of a givenb color.
- [`get_augmented_surface`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=63;drc=a63b96fa534c4513465b194e5a39aa833d37205a): Instantiate an interface extension for the given `wl_surface`.
- [`get_augmented_subsurface`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=78;drc=a63b96fa534c4513465b194e5a39aa833d37205a): Instantiate an interface extesntion for the given `wl_subsurface`.

### [`augmented_surface`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=92-187;drc=a63b96fa534c4513465b194e5a39aa833d37205a) and [`augmented_sub_surface`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=189-260;drc=a63b96fa534c4513465b194e5a39aa833d37205a)

[`augmented_surface`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=92-187;drc=a63b96fa534c4513465b194e5a39aa833d37205a) allows Client to specify the delegated composition of the surface contents.  
It has [`set_rounded_corners`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=112;drc=a63b96fa534c4513465b194e5a39aa833d37205a), [`set_destination_size`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=128;drc=a63b96fa534c4513465b194e5a39aa833d37205a), [`set_rounded_clip_bounds`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=137;drc=a63b96fa534c4513465b194e5a39aa833d37205a), [`set_background_color`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=161;drc=a63b96fa534c4513465b194e5a39aa833d37205a) and [`set_trusted_damage`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=176;drc=a63b96fa534c4513465b194e5a39aa833d37205a).  

On the other hand, [`augmented_sub_surface`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=189-260;drc=a63b96fa534c4513465b194e5a39aa833d37205a) is similar to [`augmented_surface`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=92-187;drc=a63b96fa534c4513465b194e5a39aa833d37205a) but for `wl_subsurface`.
It has [`set_position`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=201;drc=a63b96fa534c4513465b194e5a39aa833d37205a), [`set_clip_rect`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=224;drc=a63b96fa534c4513465b194e5a39aa833d37205a) and [`set_transform`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=241;drc=a63b96fa534c4513465b194e5a39aa833d37205a).

## Client side
[ui::SurfaceAugmenter](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/surface_augmenter.h) class wraps the `surface_augmenter` interface.  
[WaylandSurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_surface.h;l=45;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e) holds [`augmented_surface_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_surface.h;l=377;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e) as a privete member and [WaylandSubsurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_subsurface.h;l=24;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e) holds [`augmented_subsurface_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_subsurface.h;l=69;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e).

Client uses theses wrapper interface to set position, clip rect and other parameters on configure.  
For [WaylandSurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_surface.h;l=45;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e), for example, it calls [`augmented_surface_set_rounded_clip_bounds`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_surface.cc;l=585;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e) from [ApplyPendingState](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_surface.cc;l=446;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e) wen it appends the pending `rounded_clip_bounds` state.
For [WaylandSubsurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_subsurface.h;l=24;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e), when [WaylandFrameManger::PlayBackFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=223;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e) it calls [ConfigureAndShowSurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_subsurface.cc;l=128;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e) to configure subsurface state. It will set position and clip_rect via [`augmented_sub_surface`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/surface-augmenter.xml;l=189-260;drc=a63b96fa534c4513465b194e5a39aa833d37205a) protocol. If there is no [`augmented_subsurface_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_subsurface.h;l=69;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e) interface, it implies we don't use delegted composition so insted we use usual wl_subsurface protocol such as [`wl_subsurface_set_position`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_subsurface.cc;l=156;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e).

## Server side
[exo::wayland::AugmentedSurface](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/surface_augmenter.cc;l=38;drc=0d4bf38c8b755d4bd64aa3c26eadaeeafc9bd0ac) and [exo::wayland::AugmentedSubSurface](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/surface_augmenter.cc;l=177;drc=0d4bf38c8b755d4bd64aa3c26eadaeeafc9bd0ac) are the implementation of surface_augmenter.  

[`kSurfaceHasAugmentedSurfaceKey`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.h;l=90;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e) property is a flag whether surface is with delegated composition.  

Let's look at sub surface impl.  
On [`augmented_sub_surface_clip_rect`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/surface_augmenter.cc;l=234;drc=0d4bf38c8b755d4bd64aa3c26eadaeeafc9bd0ac) call from Client, it passes the clip rect info via [AugmentedSubSurface::SetClipRect](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/surface_augmenter.cc;l=199;drc=0d4bf38c8b755d4bd64aa3c26eadaeeafc9bd0ac), [SubSurface::SetClipRect](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/sub_surface.cc;l=52;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e) and [Surface::SetClipRect](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.cc;l=600;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e). It just passes to `pending_state_` of the surface.  
This passed `clip_rect` is applied later and used to create quad on exo by [Surface::AppendContentsToFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.cc;l=1484;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e).  
This `clip_rect` value is not transformed on Server side, so the value which Client passes will be used directly.