# Compositor Lock

## What is compositor lock?
When buffer is not yet ready, lock the compositor with the current attached buffer so that it won't show the intermediate state.  

For example, on resizing the window. Suppose the bounds which compositor thinks it's correct and that of client (lacros) holds are different, the frame buffer sent from the client is incorrect. You should not attach such buffer, instead we would like to keep the original buffer and wait for the next frame with correct bounds.  

The correct bounds will be synced by configure-ack flow, so we should lock the compositor on sending configure and unlock it on receiving ack.  

## When locked/unlocked?
*note: this is only about ShellSurface*  

[OnWindowPropertyChanged](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/shell_surface.cc;l=514;drc=a564d91c7da9f973a82851f81d3107eb2879d707)  

[OnPreWindowStateTypeChange](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/shell_surface.cc;l=551;drc=a564d91c7da9f973a82851f81d3107eb2879d707)  
This may be soon removed with my change on window state application logic.  

In ShellSurface, compositor lock instance is passed to pending_configs_ and released when AcknowledgeConfigure is called.  

## How to lock/unlock?
To lock, create CompositorLock instance and register it to CompositorLockManager. This can be done by calling Compositor::GetCompositorLock.  

You cannot explicitly unlock. The locked compositor will automatically unlocked by following two cases:
1. Destruct CompositorLock instance
2. Compositor lock is timeout.

## What happens when compositor is locked?
Cannot find where it is used expcept for DCHECK  
[BeginMainFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/compositor.cc;l=725;drc=a564d91c7da9f973a82851f81d3107eb2879d707), [DidCommit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/compositor.cc;l=776;drc=a564d91c7da9f973a82851f81d3107eb2879d707)  

However, the key is release callback.  
The release callback registered to CompositorLock is
```
base::DoNothingWithBoundArgs(host_->DeferMainFrameUpdate())
```
On constructing CompositorLock, it calculates  [cc::LayerTreeHost::DeferMainFrameUpdate](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host.cc;l=633;drc=a564d91c7da9f973a82851f81d3107eb2879d707) and it will create [ScopedDeferMainFrameUpdate](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host.cc;l=617;drc=a564d91c7da9f973a82851f81d3107eb2879d707) instance. This will be destructed when compositor lock is destructed thanks to [base::DoNothingWithBoundArgs](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/functional/callback_helpers.h;l=210;drc=a564d91c7da9f973a82851f81d3107eb2879d707) implementation.  

ScopedDeferMainFrameUpdate ctor/dtor calls [SingleThreadProxy::SetDeferMainFrameUpdate](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/single_thread_proxy.cc;l=320;drc=a564d91c7da9f973a82851f81d3107eb2879d707) and update `defer_main_frame_update_`. It is set to true when compositor is locked, false otherwise.  
`defer_main_frame_update_` value is referred as flag in two locations: in [EarlyOut in SingleThreadProxy::BeginMainFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/single_thread_proxy.cc;l=1030;drc=a564d91c7da9f973a82851f81d3107eb2879d707) and [DeferCommit SingleThreadProxy::BeginMainFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/single_thread_proxy.cc;l=1071;drc=a564d91c7da9f973a82851f81d3107eb2879d707).  

~~(Is the second one really needed?).~~ => Actually needed since `defer_main_frame_update_` may be updated while running BeginMainFrame, but why? BeginMainFrame is only posted to MainThread [ref](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/single_thread_proxy.cc;l=985;drc=a564d91c7da9f973a82851f81d3107eb2879d707) and `defer_main_frame_update_` can be only modified from SetDeferMainFrameUpdate which is guaranteed to run inside MainThread [ref](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/single_thread_proxy.cc;l=321;drc=a564d91c7da9f973a82851f81d3107eb2879d707)  

Overall, when compositor is locked, `defer_main_frame_update_` is set to true so that BeginMainFrame is skipped.


## Why SingleThreadTaskRunner?
CompositorLockManager is SingleThreadTaskRunner instead of SequencedThreadTaskRunner [ref](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/compositor_lock.h;l=37;drc=83dcfbc35ea0404b6fba0ffd0f94f2c40deb8a22)  

~~compositor_lock locks whole graphics.
Must run immediately?
However, it only used for TimeoutLocks. It's already delayed so maybe it's bearable?~~  

TaskRunner is used for CompositorLockManager and Compositor.  
One thread only for compositor.
