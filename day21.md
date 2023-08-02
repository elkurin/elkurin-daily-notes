# Widget to FrameSink

We went through some concepts in window in [Exo Surface Hierarchy](1455022) note.  
These windows are handled by someone called "WindowTreeHost" or something like "NativeWidgetAura".  
Who are they? We walk through very briefly in this note.

## Overview
The chain is like this:
view::Widget
-> aura::WindowTreeHost
-> ui::Compositor
-> cc::LayerTreeFrameSink
-> viz::CompositorFrameSinkImpl

## aura::WindowTreeHost
[aura::WindowTreeHost](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_tree_host.h) works as a bridge between a native window and ambedded RootWindow.

### On Client
On Client(Lacros), it's overridden by [DesktopWindowTreeHostPlatform](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_platform.cc;l=200;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) and [DesktopWindowTreeHostPlatform](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_lacros.cc;l=57;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83).  
It holds [NativeWidgetDelegate](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_platform.h;l=223;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) and [DesktopNativeWidgetAura](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_platform.h;l=224;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) so that the tree host can refer to widget.   The widget can be obtained from [GetWidget](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_platform.h;l=195;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83)  

DesktopWindowTreeHostLacros is constructed and owned by [DesktopNativeWidgetAura](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.cc;l=574;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) (but this is marked as raw_ptr, not uqnieu_ptr).
[DesktopNativeWidgetAura](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.h) handles top-level widgets, so WindowTreeHost exists for each widget.  
The construction is triggered on initializing native widget by [InitNativeWidget](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.cc;l=552;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) called in [Widget::Init](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/widget.cc;l=441;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83).  

### On Server
On Server(Ash), it's overridden by [AshWindowTreeHostPlatform](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/host/ash_window_tree_host_platform.cc;l=61;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83).  
This is overridden by [AshWindowTreeHostUnified](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/host/ash_window_tree_host_unified.cc;l=72;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) who creates an offscreen compositor whose texture will be copied into each display's compositor?  


## ui::Compsoitor
Compositor is responsible for compositing frames and create a single image to show to users.  
This is a concept of "composirot", but how it is impelemented in Chromium?

There is a class called [ui::Compositor](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/compositor.h). This compositor is Browser compositor who is responsible for generating final displayable form of pixels comprising a single widget's contents.  

[ui::Compositor](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/compositor.h) is created from [WindowTreeHost::CreateCompositor](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window_tree_host.cc;l=612;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) and WindowTreeHost owns it.  
Therefore, there is one Compositor for one WindowTreeHost, one Compositor for one Widget.  

ui::Compositor is also constructed on Client as well.

[DesktopWindowTreHostPlatform::Init](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_platform.cc;l=298-299;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) calls [WindowTreeHost::CreateCompositor](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window_tree_host.cc;l=612;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83).
Note that DesktopWindow* is an impelentation for Client (Lacros).  

Of course, this is created on [AshWindowTreeHostPlatform](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/host/ash_window_tree_host_platform.cc;l=69;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) as well.  

## cc::LayerTreeFrameSink
[LayerTreeFrameSink](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_frame_sink.h;l=48;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) is an interface for submitting CompositorFrames.  
This is set for each Compositor bypassing [LayerTreeHost](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host.h;l=121;drc=e90b4c68dfbd066edf8252a32e5ac85ac0d7dceb).

One Widget for One WindowTreeHost for One Compositor for One LayerTreeFrameSink.  

LayerTreeFrameSink will [SubmitCompositorFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/mojo_embedder/async_layer_tree_frame_sink.cc;l=147;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) via [compositor_frame_sink mojom API](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:services/viz/public/mojom/compositing/compositor_frame_sink.mojom;l=67;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) to viz [SubmitCompositorFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/frame_sinks/compositor_frame_sink_impl.cc;l=138;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83).

## viz::CompositorFrameSinkImpl
[CompositorFrameSinkImpl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/frame_sinks/compositor_frame_sink_impl.h;drc=fcc0cf4f5ccf071380e849bae6aa83cc4f02f8d2) is an implementation of [mojom::CompositorFrameSink](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:services/viz/public/mojom/compositing/compositor_frame_sink.mojom) to bridge [CompositorFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/quads/compositor_frame.h;l=26;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) to [CompositorFrameSinkSupport](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/frame_sinks/compositor_frame_sink_support.cc;l=543;drc=20911ffd8a0e1636801ddf303c17375fdedf9c83) and handle them by the rest.  
From here, it goes around viz sequence to produce, submit or process frames.
