# Window Tree in Views

We have digged into Exo or Wayland stack several times, but not Views.  
Let's see how window tree hierarchy are described in Views context.

## Window tree host and Content window
[DesktopNativeWidgetAura](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.h) is a top of the window hierarchy.  
This handles a top-level widgets, and other windows/widgets are children of this.  
However, this is not a "window" itelf that we imagine (the window we see as a UI).  
DesktopNativeWidgetAura owns a "window" as a direct child.

[DesktopWindowTreeHost](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/desktop_aura/desktop_window_tree_host.h;l=47;drc=37ef683d7b819722980ba25f71e8e9bcce214600) is a top of the window hierarchy.  
This class hosts whole window tree.  

[WindowTreeHost](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_tree_host.h;l=66;drc=37ef683d7b819722980ba25f71e8e9bcce214600) is another representation of the host of window tree.  
`desktop_window_tree_host_` converted into [`host_`](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.h;l=289;drc=37ef683d7b819722980ba25f71e8e9bcce214600) by AsWindowTreeHost.  
WindowTreeHost class bridges between a native window and the RootWindow.  

[`content_window_`](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.h;l=304;drc=37ef683d7b819722980ba25f71e8e9bcce214600) is aura::Window representing the root window.  
This is [appended to `host_->window()`](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.cc;l=578;drc=c1504d05a7839da026ab0a88a48109d39209c2d9) as a direct child.  
`host_->window()` returns [`window_`](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_tree_host.h;l=451;drc=37ef683d7b819722980ba25f71e8e9bcce214600) which is NativeWidget.


Each WindowTreeHost owns one ui::compositor as [`compositor_`](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_tree_host.h;l=470;drc=37ef683d7b819722980ba25f71e8e9bcce214600).  
[CreateCompositor](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_tree_host.cc;l=612;drc=37ef683d7b819722980ba25f71e8e9bcce214600) creates a compositor.  
The are within WindowTreeHost will be composited by `compositor_`.

## Correspondence against Wayland window
`host_` corresponds to platform window.  
[WindowTreeHostPlatform](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_tree_host_platform.h) is an implementation of WindowTreeHost for platform with [PlatformWindow](https://source.chromium.org/chromium/chromium/src/+/main:ui/platform_window/platform_window.h;l=34;drc=37ef683d7b819722980ba25f71e8e9bcce214600).  
PlatformWindow is for example WaylandWindow, X11Window, WinWindow....

For example, [WindowTreeHost::SetBoundsInPixels](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_tree_host.h;l=225;drc=37ef683d7b819722980ba25f71e8e9bcce214600) sets the passed bounds to `platform_window()` via PlatformWindow::SetBoundsInPixels.  
For wayland platform, it's WaylandWindow.  
This `bounds` is set to geometry via [SetWindowGeometry](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_toplevel_window.cc;l=671;drc=37ef683d7b819722980ba25f71e8e9bcce214600).  
This will eventually send `xdg_surface_set_window_geometry` wayland message.

By the way, [DesktopWindowTreeHostPlatform](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_tree_host.h;l=470;drc=37ef683d7b819722980ba25f71e8e9bcce214600) is an implementation of WindowTreeHostPlatform + [DesktopWindowTreeHost](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/desktop_aura/desktop_window_tree_host.h;l=47;drc=37ef683d7b819722980ba25f71e8e9bcce214600) for desktop aura such as lacros and linux.

`content_window_` represents a `root_surface`.  
It's accessed from DesktopWindowTreeHost, BrowserDesktopWindowTreeHost classes to manipulate the toplevel window such as OnImmersiveModeChanged, OnOveriewModeChanged, OnTooltipShownOnServer, or on UpdateFrameHints.

## Window Bounds
`content_window_` represents a toplevel window, so its bounds is the window size and its offset from the screen.  
DesktopNativeWidgetAura, on the other hand, does not have to be the same as the `content_window_` in theory.  
The size of DesktopNativeWidgetAura (= WindowTreeHost) specifies the area that compositor works.  

