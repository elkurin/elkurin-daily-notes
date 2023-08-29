# Decoration Insets in Linux

Lacros is based on Linux, but has a different implementation for some places.  
In the graphic context, Linux using X11 as a backend works very differently from Lacros.  
We go through how the client side decoration bounds is handled.

## Client side decoration
Linux widely draws shadow of the frame on client side while Lacros does not.  
We can see its condition from [SupportsClientFrameShadow](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/frame/browser_desktop_window_tree_host_linux.cc;l=180;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa).  
[CanSetDecorationInsets](https://source.chromium.org/chromium/chromium/src/+/main:ui/platform_window/platform_window.cc;l=62;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa) is overridden by [WaylandToplevelWindow](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_toplevel_window.cc;l=356;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa), which returns `xdg_wm_base` existence, and [X11Window](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/x11/x11_window.cc;l=1061;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa) depending on the window manager types.

This "shadow" is one of the decoration.  
On Linux, Client side decoration is widely supported.

## Updating insets on Linux
[BrowserDesktopWindowTreeHostLinux::UpdateFrameHints](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/frame/browser_desktop_window_tree_host_linux.cc;l=185;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa) calculates a shadow inset and set it as decoration insets.

Here's the summary.
```cpp=
void BrowserDesktopWindowTreeHostLinux::UpdateFrameHints() {
  auto* window = platform_window();
  float scale = device_scale_factor();
  ...;
  
  if (SupportsClientFrameShadow()) {
    gfx::Insets insets = view->MirroredFrameBorderInsets();
    ...;
    const gfx::Insets insets_px = gfx::ScaleToCeiledInsets(insets, scale);
    window->SetDecorationInsets(showing_frame ? &insets_px : nullptr);
    
    gfx::Rect input_bounds(widget_size);
    input_bounds.Inset(insets + view->GetInputInsets());
    input_bounds = gfx::ScaleToEnclosingRect(input_bounds, scale);
    window->SetInputRegion(showing_frame
                               ? absl::optional<gfx::Rect>(input_bounds)
                               : absl::nullopt);
  }
  ...;
}
```

If client frame shadow is supported, Linux draws shadow on the client side.  
It gets [BrowserFrameViewLinux::MirroredFrameBorderInsets](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/frame/browser_frame_view_linux.cc;l=30;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa) as an inset for the shadow.  
This impl is simply returns [kFrameBorderThickness](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/frame/opaque_browser_frame_view_layout.h;l=37;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa) which is `4` or calculates from [GetRestoredFrameBorderInsetsLinux in browser_frame_view_paint_utils_linux.cc](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/frame/browser_frame_view_paint_utils_linux.cc;l=56;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa).  
This seems to be used for most cases.

THe calculated inset is set to platform window by [SetDecorationInsets](https://source.chromium.org/chromium/chromium/src/+/main:ui/platform_window/platform_window.cc;l=66;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa).  
We will check this in the next section.

The latter half of the above code is updating the input region.  
Input region is an area that absorbs the input event.  
When you click somewhere, or touch somewhere, or type something, the input event is dispatched to the correct target.  
Input region specifies such covered area.

## SetDecorationInsets
SetDecorationInsets is implemented on 2 platforms, Wayland and X11.

### Wayland
As for [WaylandWindow](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_window.cc;l=543;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa), it updates `frame_insets_px_` value.  
[`frame_insets_px_`](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_window.h;l=566;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa) is an optional value representing mrgins between edges of the surface and the window geometry.  
Window geometry should contain only the content itself while the compositor should contain all rendered frame such as shadow. If not, they cannot be painted.  
The set inset is refered from [SetWindowGeometry](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_toplevel_window.cc;l=671;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa) to subtract the inset from `size_dip` to identify the geometry.

### X11
As for [X11](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/x11/x11_window.cc;l=1080;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa), the inset param is set by [SetArrayProperty](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/x/xproto_util.h;l=88;drc=f2732aef3e95dca51f5d70c1242f0c7614a0b9aa) only when PlatformWindowStte is kNormal or kUnknown.  
The property change is sent to the server via xproto.cc.  
Looks like the server is calculting the bounds?  
This is same for SetInputRegion.


