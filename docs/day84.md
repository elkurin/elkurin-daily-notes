# Decoration in Chromium Wayland

In Chromium Wayland, there is a concept called decoration.  
For example, the shadow around the window is a decoration.  
This is a reading note related to decoration codes for Lacros and Linux.

## Decoration Insets
Decoration insets represents the insets between the surface and the geometry.  
The surface is an area of the toplevel window. The geometry is an area that is visible to the user.  
This value is set when the decoration is done by the client, not the server.

The decoration insets can be set from [SetDecorationInsets](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_window.cc;l=543;drc=0aa601c51f8afbcbe773d0b57e6a3cf9f4b41b22).  
The passed value is set to [`frame_insets_px`](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_window.h;l=566;drc=16f579bd5f425f27c30816cac4b06508bcb3e5c8) as pixel coordinates.  

Currently, this decoration insets is only used for linux platform from [browser_desktop_window_tree_host_linux.cc](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/frame/browser_desktop_window_tree_host_linux.cc;l=209;drc=16f579bd5f425f27c30816cac4b06508bcb3e5c8).  
Also, this insets is ignored when [DisableClientSideDecorationsForTest](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/wayland_utils.cc;l=46;drc=16f579bd5f425f27c30816cac4b06508bcb3e5c8) is called. This method create scoped instance that flips the global param [`g_disallow_setting_decoration_insets_for_testing`](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_toplevel_window.cc;l=46;drc=16f579bd5f425f27c30816cac4b06508bcb3e5c8).  Though I was curious what kind of tests requires client side decorations to be disabled, this class seems to be not used from anywhere right now.


### Input Region
On setting decoration insets, we also need to adjust input regions.  
Input region represents the area where input events such as key events, mouse events... are absorbed by.  
When the event is dispatched, we need to find who is targetted by the event. Input region is the key concept to calculate the target.  
Even if we expand the image area by decoration insets, we cannot absorb the event dispatched to the expanded area as long as the input regions stay as same as the toplevel window.  

In browser_desktop_window_tree_host_linux.cc [SetInputRegion](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/frame/browser_desktop_window_tree_host_linux.cc;l=215;drc=16f579bd5f425f27c30816cac4b06508bcb3e5c8) is called just below SetDecorationInsets as expected.

## How decoration insets is used
The decoration insets is used to calculate geometry inside [SetWindowGeometry](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_toplevel_window.cc;l=690;drc=0aa601c51f8afbcbe773d0b57e6a3cf9f4b41b22).  
In WaylandToplevelWindow, the origin of geometry is (0, 0), so it just expands the size_dip by decoration insets.  

So, on wayland, the decoration is expected to be inside geometry and it's already drawn on the client side. There is no special protocol to specify the decoration area from the client to the server.

## Decoration protocol
Still there is a protocol related to decoration.  

[set_decoration](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/aura-shell.xml;l=956;drc=c34fbe3e0f2337304dad059e056db4b6cac78975) is defined in aura-shell.xml.  
This protocol specifies which frame type the aura surface is.  
The parameter is [decoration_type](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/aura-shell.xml;l=965;drc=c34fbe3e0f2337304dad059e056db4b6cac78975) to specify the frame type.  
There are 3 types, "NONE" for no frame, "NORMAL" for a caption with shadow and "SHADOW" for shadow only.

But this protocol is not used for toplevel window right now.  
That's probably because we now decorate on the client side, not the server side.
