# Graphic Pipeline on Chrome Compositor (cc)

Chrome Compositor a.k.a. cc is a compositor used in both the renderer and the browser. Its client is Blink for the renderer, and UI and Android browser compositor for the browser.  
It is implemented in [cc/](https://source.chromium.org/chromium/chromium/src/+/main:cc/) directory.

This note aims to capture the rough picture of graphic pipeline by folowing cc's architecture.

## What does cc do?

cc is responsible for
1. Taking painted inputs from its embedder (ui/compositor, blink...)
2. Figuring out whether they appear on screen and where if so
3. Rasterizing, decoding and animating images from the painted input into GPU textures
4. Forwarding those textures on to the display compositor as CompositorFrame
5. [extra] Handling pinch/scroll gestures responsibly (without involving Blink)

What are these?

### Pipeline With Examples
Here's a general explanation on Graphics.

Image displayed on a monitor is usually a complex combination of multiple images and boundaries.  

For example, look at this example frame:

![](https://hackmd.io/_uploads/HkgRFWrF3.png)

There can be many windows opened and they may overlay each other.  
Or the video frame may be partially invisible as it's scrolled out.  
Or you can drag a window at the bottom so that the window is out of the screen boundary. In this case, Settings window at the bottom is not completely hidden by Application list bar but it's a bit transparent.  
Also, there are shadows around the window.  
It's not included in this figure, but you can also hover upon a link to see a tooltip, right click somewhere to open context menu, ot resize the window...

To deal with this, we don't create a image at once.  
We first create individual seperated images, for example for each window.
Each application (browser, Android app...) are asked to create a painted inputs, and they send the produced image to cc (**Step 1**).  

After cc gathers painted inputs, cc figures out which images or which part of the images are visible. Also, it calculates the position to show on the display. Each client knows what is where on their local coordinates, but not on display coordinates. Each coordinates may have different origins and different scales. cc is responsible for translating those coordinates into primary coordinates used in the screen. (**Step 2**)

Next cc manipulates taken painted inputs into GPU textures.  
Rasterization is an operation to transforms an image into a certain scale bitmap. We may need to fill in the gap when we make it larger, or we squash it to make it smaller. For example, in the overview mode (below picture), the drawn images instantly become smaller. This is not drawn from scratch on entering overview mode, but instead we "rasterize" the existing image to scale them down. (NOTE: this feature is not, or not yet available for some platform/settings).  
![](https://hackmd.io/_uploads/ryODzMBYn.png)

We may also need to animate the window. For example, on maximizing window it gradually changes the size smoothly instead of instant change. This is called "animating".  
cc goes through these operations and generate GPU textures (**Step 3**).

By the way, what is "GPU texture"?  
In graphics context, we use "texture" as a raw image. For example in 3D modeling, we have a technique called Texture Mappng. We generates a wallpaper (this is called texture) and attaches it to the model such as a human. This way you can create a image of human wearing T-shirts with more reality.  
(Found an interesting article about [Tile-Based Texure Mapping](https://developer.nvidia.com/gpugems/gpugems2/part-ii-shading-lighting-and-shadows/chapter-12-tile-based-texture-mapping) will read it later.)  
In cc's context, GPU texture is jsut a produced image.  
This texture is forwarded to display compositor as CompositorFrame. This is the output cc produces (**Step 4**).  
 
[viz::CompositorFrame](https://source.chromium.org/chromium/chromium/src/+/main:components/viz/common/quads/compositor_frame.h;l=26;drc=8ba1bad80dc22235693a0dd41fe55c0fd2dbdabd) consists of metadata such as device scale, color space and size. and an ordered set of render passes.  

## Flow and Threadings on cc
Let's look at the whole flow showed above as one line:  
![](https://hackmd.io/_uploads/BJnX2NrK3.png)
(Cite from [How cc Works](https://source.chromium.org/chromium/chromium/src/+/main:docs/how_cc_works.md))

This flow is scheduled by [cc::Scheduler](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/scheduler/scheduler.h)  

As you can see, cc handles from painting to submitting compositor frame.  
There are two threads here: Main Thread and Compositor Thread (and Raster thread? what's this? TODO)  
This "thread" is more of a conceptual thing and it may not be an independent thread. This could be single threaded.  
Currently, ui::compositor and cc wrapper uses single threaded cc.  
Also Blink renderer [looks to be calling multi-threaded cc](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/blink/renderer/platform/widget/compositing/layer_tree_view.cc;l=114-121;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) for non-tests, but AFAIK no one seems to be creating [`g_compositor_thread_scheduler`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/blink/renderer/platform/scheduler/worker/compositor_thread_scheduler_impl.cc;l=22;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) so that [LayerTreeView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/blink/renderer/platform/widget/compositing/layer_tree_view.cc;l=91;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) does not receive `compositor_thread` and ends up creating single-threaded cc.  

Main Thread requests Compositor Thread to paint something, for example animation, then Compositor Thread triggers BeginMainFrame (explained in [ChromeOS compositing - How Frame is Produced](/docs/day4.md)). After the frame is produced, Main Thread norifies Compositor Thread that they are ready to commit, and then Compositor Thread schedule commit action. During the last stage, Main Thread is blocked.  

### Example: Layer - LayerImpl
Let's look at the flow with codes. To make it clearer, we go through multi-threaded flow.  

Suppose the client requests to change background color.  
In Main Thread, [Layer::SetBackgroundColor](https://source.chromium.org/chromium/chromium/src/+/main:cc/layers/layer.cc;l=532;drc=69e6dc49684309c8b375c4dcd724c6ae61878ecd) updates `inputs_.background_color` accordingly to the requested value, then calls [Layer::SetNeedsCommit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer.cc;l=200;drc=255b4e7036f1326f2219bd547d3d6dcf76064870).  
[LayerTreeHost::SetNeedsCommit] requests that the next main frame update performs a full commit synchronization to Compositor Thread bypassing [Proxy::SetNeedsCommit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/proxy.h;l=49;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) and [ProxyMain::SendCommitRequestToImplThreadIfNeeded](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/proxy_main.cc;l=802;drc=255b4e7036f1326f2219bd547d3d6dcf76064870).  
It comes to Compositor Thread from [ProxyImpl::SetNeedsCommitOnImpl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/proxy_impl.cc;l=231;drc=255b4e7036f1326f2219bd547d3d6dcf76064870). [Scheduler::SetNeedsBeginMainFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/scheduler/scheduler.cc;l=142;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) immediately sends [ProxyImpl::ScheduledActionSendBeginMainFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/proxy_impl.cc;l=703;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) request back to Main Thread.  
Main Thread starts [ProxyMain::BeginMainFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/proxy_main.cc;l=134;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) cycle.
When the cycle has completed, Main Thread posts task [ProxyImpl::NotifyReadyToCommitOnImpl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/proxy_impl.cc;l=319;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) to Compositor Thread with [blocking main thread](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/proxy_main.cc;l=450-452;drc=255b4e7036f1326f2219bd547d3d6dcf76064870).  
Compositor Thread updates the scheduler state machine to COMMIT and call [ProxyImpl::ScheduledActionCommit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/proxy_impl.cc;l=764;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) to shedule a commit. This will start the flow after the commit in above figure by [LayerTreeHostImpl::BeginCommit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host_impl.cc;l=634;drc=255b4e7036f1326f2219bd547d3d6dcf76064870). From here, Compositor Thread goes through Commit -> Raster -> Activate -> Submit -> (viz) Aggregate flow. When it's all done, it comes back to Main Thread by [ProxyMain::DidCompleteCommit](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/proxy_main.cc;l=480;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) and finish!  
 
## Impl?
Many files in cc/ contains a word "impl". We usually use impl as a file which implementes an abstract class. For example, [browser_process.h](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/browser_process.h) includes an abstract class [`BrowserProcess`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/browser_process.h;l=111;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) and its implementation is in [browser_process_impl.h](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/browser_process_impl.h) [`BrowserProcessImpl`](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/browser_process_impl.h;l=93;drc=4332a2f3be882f937dcb8ae8e205d910d786b20a).  
However, it's different on cc, such as [layer_impl.h](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer_impl.h;l=70;drc=255b4e7036f1326f2219bd547d3d6dcf76064870) implements class called LayerImpl but this does not override any class. What's this?  

Historycally, we had a thread called Impl thread. The class including "impl" tag implies that it used to run on impl thread.  
We can still find [`IsImplThread()`](https://source.chromium.org/chromium/chromium/src/+/main:cc/trees/layer_tree_host.h;l=203;drc=072f32b089c59dc00d644c15a9ee1f29bdcec913) together with [`IsMainThread()`](https://source.chromium.org/chromium/chromium/src/+/main:cc/trees/layer_tree_host.h;l=202;drc=072f32b089c59dc00d644c15a9ee1f29bdcec913).  

Now impl thread is renamed to "compositor thread".  
By the way, we no longer seperate main thread and compositor thread as a physical thread, but just as a conceptual thread, so it's not even a thread.
