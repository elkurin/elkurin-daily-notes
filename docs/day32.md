# Immersive Mode on ChromeOS

"Immersive Mode" is a window state in ChromeOS which only shows web content inside one tab. Unlike "Maximized" state, it won't show tabs nor application shelf like this:
![](https://hackmd.io/_uploads/SyZTWbDdh.png)


## Immersive Mode Behavior
You can enter Immersive Mode by pressing F11 key or selecting fullscreen mark in "Customize and control Google Chrome" menu. In ChromeOS, instead of F11, you can press fullscreem marked key at the spot of F4 in other platform.

While the window is in immersive mode, we cannot see the tab list nor app list, but it's inconvenient if we cannot see them at all without exiting immersive mode.  
Therefore, we have gestures to resolve it.  
By swiping down the top area, the tab list is shown.  
By swiping up the bottom area, the app shelf is shown.  
These are called as "Revealed".

## Enter/Exit Immersive Mode
Let's look at the entering path.  
Starts from [ToggleFullscreenMode](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/browser_commands.cc;l=1871;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) in browser commands.  
Then triggers [FullscreenController::ToggleBrowserFullscreenMode](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/exclusive_access/fullscreen_controller.cc;l=87;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) to [ExclusiveAccessContext::EnterFullscreen](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/exclusive_access/exclusive_access_context.h;l=44;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061).  
This is overridden by [BrowserView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/frame/browser_view.cc;l=1726;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061).  
[Widget::SetFullscreen](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/widget.cc;l=917;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) will set the frame into immersive mode. For Lacros, it triggers [WaylandToplevelWindow::SetFullscreen](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_toplevel_window.cc;l=206;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) and send the window state to Ash via wayland.  

Exit path is almost the same by setting `fullscreen` boolean to false instead of true.

When ImmersiveMode is changed, it notifies [BrowserDesktopWindowTreeHostLacros::OnImmersiveModeChanged](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/frame/browser_desktop_window_tree_host_lacros.cc;l=227;drc=a173d81e78fe4c609008d0c848f37460fe01dc45) and update the show status of the top toolbar in [BrowserCommandController::UpdateCommandsForFullScreenMode](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/browser_command_controller.cc;l=1654;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) and [Browser::UpdateBookmarkBarState](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/browser.cc;l=3073;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061).  
Whether location bar should be shown or not is handled by WindowFeature.  
[NormalBrowserSupportsWindowFeature](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/browser.cc;l=2940;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) returns boolean to imply whether the given WindowFeature is supported.  
Tabstrip, toolbar and locationbar is not supported when [ShouldHideUIForFullscreen](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/frame/browser_view.cc;l=1810;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) is true and its condition is whether Immersive Mode is enabled.

## ImmersiveFullscreenController
[ImmersiveFullscreenController](https://source.chromium.org/chromium/chromium/src/+/main:chromeos/ui/frame/immersive/immersive_fullscreen_controller.h) is responsible for controlling immersive mode state itself and behavior.  
It triggers immersive mode updates on window property change, bounds change and other events.

Immersive Mode availability is stored as [`enabled_`](https://source.chromium.org/chromium/chromium/src/+/main:chromeos/ui/frame/immersive/immersive_fullscreen_controller.h;l=266;drc=89e1f63c97ae8c66ff1b8de4fbf564978349ac70) which is updated when window property [kImmersiveIsActive](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/base/window_properties.cc;l=39;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) is set.

There is [ImmersiveModeControllerChromeOS](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/frame/immersive_mode_controller_chromeos.cc) as an implementation of ChromeOS.

### Immersive Revealed Lock
To handle swipe behavior on immersive mode correctly, we introdue [ImmersiveRevealedLock](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/frame/immersive/immersive_revealed_lock.h;drc=e4714ce987b39d3207473e0cd5cc77fbbbf37fda).  
Its inner implementation is [LockRevealedState/UnlockRevealedState](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/frame/immersive/immersive_fullscreen_controller.cc;l=265-280;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061). It uses [`revealed_lock_count_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/frame/immersive/immersive_fullscreen_controller.h;l=271;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) as a counter to manage the state.

## Other notes
Here, writing down annotations of view parts.  

[ToolbarView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/toolbar/toolbar_view.h;l=72;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061): View of toolbar (at the top of the window)  
[TabStrip](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/tabs/tab_strip.h): Represents tab strip model.  
[TabStripRegionView](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/frame/tab_strip_region_view.h): Container for the tabstrip and other views sharing space with it.  
[TabStripScrollContainer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/tabs/tab_strip_scroll_container.h): Tab strip scroll horizontally.  
[TopContainerView](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/frame/top_container_view.h): Container for the BrowserView's tab strip, toolbar and bookmark bar. As for ChromeOS immersive mode, it stacks on top of other views so that we can see it on sliding down.  
[TopControlsSlidControllerChromeOS](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/frame/top_controls_slide_controller_chromeos.h): Implements the Android-like slide behavior of the browser top controls for ChromeOS.  
