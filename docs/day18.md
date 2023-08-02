# Chromium Window Destruction Flow
In Chromium, there are many classes representing the concept of "window", or closely related to it.  
[Widget](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/widget.h), [aura::Window](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window.h), [NativeWidget](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/native_widget.h), [PlatformWindow](https://source.chromium.org/chromium/chromium/src/+/main:ui/platform_window/platform_window.h), [WaylandWindow](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_window.h), [WindowTreeHost](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_tree_host.h), [DesktopNativeWidgetAura](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.h), [DesktopWindowTreeHost](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/widget/desktop_aura/desktop_window_tree_host.h)...  
This is a self-note to summarize how they are destructed in order.

## Who destroys Window?
Chrome browser can be killed from both client or server.  
When the user clicks "Close" button at top-right of the window, the client dispatches window destruction event.  
On the other hand, you can close the window from shortcut key, and this case Server is the one who dispatches window destruction event.  
In both scenario, the window representation on both Client and Server side must be abandoned correctly.

## From Client
First, let's look at the flow on closing window from Client.  

- [chrome::CloseWindow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/browser_commands.cc;l=852;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) is an API to close window.
- [BrowserView::Close](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/frame/browser_view.cc;l=1286;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) will be called for ordinal Chrome window.
- [Widget::Close](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/widget.cc;l=772;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) is dispatched. From here, Widget destruction starts.
- [DesktopNativeWidgetAura::Close](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.cc;l=904;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) is called by overriding [NativeWidgetPrivate](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/native_widget_private.h;l=183;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) for Lacros Client.
- [DesktopWindowTreeHostPlatform::Close](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_platform.cc;l=350;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) calls [aura::Window::Hide](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window.cc;l=362;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) and [WindowTreeHost::Hide](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window_tree_host.cc;l=389;drc=f2c32198ce5251791dc0a79ad63404509c22fee1), and then [PostTask](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_platform.cc;l=367;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) [DesktopWindowTreeHostPlatform::CloseNow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_platform.cc;l=372;drc=f2c32198ce5251791dc0a79ad63404509c22fee1).
  - [aura::Window::Hide](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window.cc;l=362;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) [SetVisibilityInternal](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window.cc;l=985;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) to false. This only changes the visibility while the window itself stays alive.
  - [WindowTreeHost::Hide](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window_tree_host.cc;l=389;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) calls [PlatformWindow::Hide](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/platform_window/platform_window.h;l=44;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) bypassing [WindowTreeHostPlatform::HideImpl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window_tree_host_platform.cc;l=109;drc=f2c32198ce5251791dc0a79ad63404509c22fee1). On Lacros, it's overridden by [WaylandWindow::Hide](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_window.cc;l=305;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) which calls [WaylandFrameManager::Hide](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=802;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) and clears FrameCallback, `submitted_buffers_`, `pending_frames_` and [MaybeProcessSubmittedFrames](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_frame_manager.cc;l=643;drc=f2c32198ce5251791dc0a79ad63404509c22fee1). Frame with hidden window is produced and probably sent to Server going through viz (Not confirmed, it could be just skipped?)
  - [DesktopWindowTreeHostPlatform::CloseNow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_platform.cc;l=372;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) will actually close the window just now. It does
    - PlatformWindow::PrepareForShutdown
    - ReleaseCapture()
    - clild->CloseNow()
    - DestroyCompositor()
  - [PlatformWindow::Close](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/platform_window/platform_window.h;l=45;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) is called from [DesktopWindowTreeHostPlatform::CloseNow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_platform.cc;l=372;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) which is overriden by [WaylandWindow::Close](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_window.cc;l=374;drc=b254fd126e05f71848bd58aba7c46e649794807f)
  - [DesktopWindowTreeHostPlatform::OnClosed](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_window_tree_host_platform.cc;l=841;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) is called and does followings:
    - ```cpp
      desktop_native_widget_aura_->OnHostWillClose();
      SetPlatformWindow(nullptr);
      desktop_native_widget_aura_->OnHostClosed();
      ```
  - [`SetPlatformWindow(nullptr)`](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_tree_host_platform.h;l=73;drc=a361313206107398cab3d66afc21e019c116861b) goes to
    - [WaylandWindow::~WaylandWindow()](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_window.h;l=71;drc=b254fd126e05f71848bd58aba7c46e649794807f)
    - [~zaura_surface()](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_window.h;l=530;drc=b254fd126e05f71848bd58aba7c46e649794807f) this will disconnect with Server
    - [AuraSurface::~AuraSurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/wayland/zaura_shell.cc;l=350;drc=f2c32198ce5251791dc0a79ad63404509c22fee1) is called when zaura_surface() connection is lost. It destroyes Surface nd triggers SurfaceObserver::OnSurfaceDestroyed, and destroys whole hierarchy.
    - Surfaces and windows on Server is not explicitly closed from Client, but instead destructed by losing connection IIUC.
      - TODO(elkurin): How ~Surface() is called? Who owns surface?
  - [DesktopNativeWidgetAura::OnHostClosed](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.cc;l=330;drc=f2c32198ce5251791dc0a79ad63404509c22fee1)

    ```cpp=
    tooltip_controller_.reset();
    ...
    desktop_window_tree_host_ = nullptr;
    host_.reset();
    content_window_ = nullptr;
    ```
    WindowTreeHost is destructed at this point.
  - [Widget::OnNativeWidgetDestroyed](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/widget.cc;l=1594;drc=f2c32198ce5251791dc0a79ad63404509c22fee1)
    ```cpp=
    observer.OnWidgetDestroying();
    ...
    native_widget_.reset();
    ```


## From Server
On the other hand, when server destroyes window it goes like this:

- [Winwdow::~Window()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window.cc;l=184;drc=f2c32198ce5251791dc0a79ad63404509c22fee1)
- [ShellSurfaceBase::OnWindowDestroying](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/shell_surface_base.cc;l=1336;drc=f2c32198ce5251791dc0a79ad63404509c22fee1)
- [WaylandToplevel::OnClose](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/xdg_shell.cc;l=282;drc=761c0539e4d37a69a3f6140b00c5fa40ed17699a) is set to `close_callback_`
- xdg_toplevel_send_close
- *... wayland ipc ...*
- xdg_toplevel_listener [CloseTopLevel](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/xdg_toplevel_wrapper_impl.cc;l=107;drc=f2c32198ce5251791dc0a79ad63404509c22fee1)
- [XDGToplevelWrapperImpl::CloseTopLevel](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/xdg_toplevel_wrapper_impl.cc;l=357;drc=f2c32198ce5251791dc0a79ad63404509c22fee1)
- [WaylandWindow::OnCloseRequest](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_window.cc;l=646;drc=b254fd126e05f71848bd58aba7c46e649794807f)
- [WindowTreeHostPlatform::OnCloseRequest](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window_tree_host_platform.cc;l=241;drc=f2c32198ce5251791dc0a79ad63404509c22fee1)
- [WindowTreeHost::OnHostCloseRequested](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window_tree_host.cc;l=704;drc=f2c32198ce5251791dc0a79ad63404509c22fee1)
- [DesktopNativeWidgetAura::OnHostCloseRequested](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/desktop_aura/desktop_native_widget_aura.cc;l=1446;drc=f2c32198ce5251791dc0a79ad63404509c22fee1)
- [Widget::Close](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/widget.cc;l=772;drc=f2c32198ce5251791dc0a79ad63404509c22fee1)
- same path explained above
