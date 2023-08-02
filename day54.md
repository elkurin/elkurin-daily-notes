# Dragging in Chrome

Dragging event has some complex code stacking.  
There is a seperate protocl for drag-and-drop a.k.a DnD.  
Wayland Protocol book also has a seperated page for [Drag & drop] but it looks like empty somehow...?  

Anyway, we use wayland drag-and-drop protocol for dragging tabs, windows and items.  

## Browser UI Views
They are two usages of drag-and-drop.

### Tab drag
Used when you drag tab or whole window.

[TabDragController::Drag](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/tabs/tab_drag_controller.cc;l=567;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) dispatches the dragging event.  
While the tab is dragged, they are handled as [TabSlotView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/tabs/tab_slot_view.h;l=14;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588). The target tabs are collected from [GetViewsMatchingDraggedContents](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/tabs/tab_drag_controller.cc;l=1741;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) and passed to [TabDragContextImpl::StartedDragging](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/tabs/tab_strip.cc;l=533;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588).  
The evene is passed through the below path:  
[BrowserTabStripController::OnStartDragging](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/tabs/browser_tab_strip_controller.cc;l=484;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588)  
-> [BrowserFrame::SetTabDragKind](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/frame/browser_frame.cc;l=408;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588)  
-> [DesktopBrowserFrameLacros::TabDraggingKindChanged](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/frame/desktop_browser_frame_lacros.cc;l=38;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588)  
-> [BrowserDesktopWindowTreeHostLacros::TabDraggingKindChanged](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/frame/browser_desktop_window_tree_host_lacros.cc;l=166;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588)  
-> [WaylandToplevelWindow::StartWindowDraggingSessionIfNeeded](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_toplevel_window.cc;l=764;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588)  
-> [WaylandWindowDragController::StartDragSession](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_window_drag_controller.cc;l=120;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588)

Their 3 types of [TabDragKind](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/frame/browser_frame.h;l=49-59;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588), `kNone` representing no drag is active, `kTab` representing one or more but not all tabs within a window are being draged, and `kAllTabs` representing all of the tabs in a window are being dragged. `kAllTabs` implies that it's dragging the window.  
Both of them are handled in the same protocol.  

### Item drag
Used when you drag an item such as an image to for example a download box or folder.  
[DragDownloadItem](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/download/drag_download_item_aura.cc;l=26;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) is one of the usage.  
It goes to [WaylandWindow::StartDrag](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_window.cc;l=255;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) and then [WaylandDataDragController::StartSession](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_data_drag_controller.cc;l=119;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588).  
This time, unlike tab dragging, we set mime type using [WaylandExchangeDataProvider](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_exchange_data_provider.h;l=16;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) and construct a different mime type depending on the content to drag.  

## Wayland Platform Implementation
Let's see how it is handled on ozone/wayland.  

It's controlled by [WaylandWindowDragController](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_window_drag_controller.h) on wayland.

Tab drag is conreolled by state machine:  
```cpp=
enum class State {
  kIdle,       // No DnD session nor drag loop running.
  kAttached,   // DnD session ongoing but no drag loop running.
  kDetached,   // Drag loop running. ie: blocked in a Drag() call.
  kDropped,    // Drop event was just received.
  kCancelled,  // Drag cancel event was just received.
  kAttaching,  // About to transition back to |kAttached|.
};
```

Initially, `state_` is set as `kIdle`.  

On dragging [StartDragSession](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/wayland_window_drag_controller.cc;l=120;drc=b254fd126e05f71848bd58aba7c46e649794807f) is called as explained in above section, and the state is updated to `kAttached`.  
Then we create [WaylandDataSource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_data_source.h;l=39;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) from [WaylandDataDeviceManager](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_data_device_manager.h;l=19;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) and set Drag-and-Drop action to [kDndActionWindowDrag](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_window_drag_controller.cc;l=64;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588). FYI: DnD is a short word for drag-and-drop.  
While we are dragging, the window uses custom mime type `chromium/x-window` defined by [kMimeTypeChromiumWindow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_window_drag_controller.cc;l=61;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588).  
Then [StartDrag](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_data_device.cc;l=36;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) from data device. [WaylandDataDevice](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_data_device.h) class provides access to inter-client data transfer mechanisms which wraps wl_data_device protocol such as `wl_data_device_start_drag`.  
[StartDrag](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_data_device.cc;l=36;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) calls `wl_data_device_start_drag` protocol. (For window drag controller, [DrawIcon](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_window_drag_controller.cc;l=207;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) is done by buffer manager.)  

Server side impl in Exo is in [wl_data_device_manager.cc](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/wl_data_device_manager.cc).   
Let's look at the drag event triggered from mouse input from here.  
`wl_data_device_start_drag` goes to [Seat::StartDrag](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/seat.cc;l=122;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588).  
It obtains `cursor_location` inside Ash from [Env::GetLastPointerPoint](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/env.cc;l=157;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) and then creates [DragDropOperation](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/drag_drop_operation.cc;l=157;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) with `cursor_location` as drag start point.  
[DragDropOperation ctor](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/drag_drop_operation.cc;l=169;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) is a main part. It [ScheduleStartDragDropOperation](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/drag_drop_operation.cc;l=356;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) which post [StartDragDropOperation](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/drag_drop_operation.cc;l=374;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) task to sequenced task runner.  

## Extended drag source
[ExtendedDragSource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_window_drag_controller.cc;l=76;drc=636048a55ca7e2e35d5a15ba10d64b03dfe8c588) may be supported on new enough version.  
It can use `zcr_extended_drag_source_v1` protocol and the server side impl is in [zcr_extended_drag.cc](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/zcr_extended_drag.cc) and the protocol is designed in [extended-drag-unstable-v1.xml](https://source.chromium.org/chromium/chromium/src/+/main:third_party/wayland-protocols/unstable/extended-drag/extended-drag-unstable-v1.xml).  
This extends the existing Wayland drag-and-drop protocol such as making toplevel shell surfaces draggable or snappable.  
It is required for Chromium-like tab tragging UX where the user is able to drag a tab or any other kind of UI piece out of its original window into a new surface.  
