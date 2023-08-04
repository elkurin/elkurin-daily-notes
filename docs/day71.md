Let's understand the overview of how Chromium graphics works.

## Server vs Client
When Browser runs on OS, we see visible images. These graphics depend on platform to draw. Each platform has a specific window managing protocol and Chrome switches the backend impelmentation depending on the platform.  
If the current platform uses X11, then it switches to X11 implementation. If it's Wayland, it switches to Wayland... so on.

The backend implementation differs for each platform and [Ozone](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/ozone_overview.md) is one of a platform abstraction layer to connect between Aura and window system.  
[PlatformWindow](https://source.chromium.org/chromium/chromium/src/+/main:ui/platform_window/platform_window.h) is an abstract window class for Ozone and each platform overrides this class such as [X11Window](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/x11/x11_window.h;l=45;drc=39d7898be01c003ba2ac70b5ac7bac27a843a385), [WaylandWindow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_window.h;l=66;drc=39d7898be01c003ba2ac70b5ac7bac27a843a385), [WinWindow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/platform_window/win/win_window.h;l=22;drc=39d7898be01c003ba2ac70b5ac7bac27a843a385), [DrmWindowHost](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/drm/host/drm_window_host.h;l=39;drc=39d7898be01c003ba2ac70b5ac7bac27a843a385)...  

TODO(elkurin): How is MacOS implemented?  


### ChromeOS and Wayland
On ChromeOS, Lacros (the browser) is a client and Ash is Server.  
Here, we uses [wayland](https://wayland-book.com/) as a window server protocol to interact with each other simiarly to Linux platform.  

Wayland provides a basic package of event handling and buffer submissions.  
On ChromeOS, we extends the protocol for our usage.
For example, [aura-shell.xml](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/aura-shell.xml) defines the extended interface for aura shell.

### What they do
How and where to divide a role of Server and Client is decided for each platform independently.  
Moreover, the boundary is updated frequently for one platform.  
For example on ChromeOS, tooltip window used be produced on Lacros side and the buffer was sent to Server, but now it updated not to produce on Lacros side but delegate it to Server (by me) for performance reasons.  
There are many such examples.

Let's look at what [PlatformWindow](https://source.chromium.org/chromium/chromium/src/+/main:ui/platform_window/platform_window.h) has.  
It has Show/Hide/Close as very basic methods to manipulate. It also has Maximize/Minimize/Restore/SetFullscreen to manipulate the window state.  
Another important property is bounds. The bounds specifies where the window is located with x,y and width,height in some coordinates.  
Cursor and input forucs is also a target here.

We can assume these are the expected roles for Client side to handle.

## GPU and Browser
There is another separation which is GPU process and Browser process.  
Browser process uses CPU while GPU process uses GPU as the name says.  
By the way, I found a great metaphor for GPU vs CPU while I was reading [GPUを支える技術](https://amzn.asia/d/3ealGF6).
> CPU is a sport car. GPU is a buss.

Now that GPU becomes common and most computers contains such units, Chrome also has a GPU process and use GPU to render, not CPU as it used to do.  
This makes rendering more smoother and faster (but probably with memory...)

### ChromeOS GPU process
Let's focus on ChromeOS case.  
Both Lacros and Ash have their own GPU process.  

[/gpu](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/gpu/) contains code run in client-side GPU process and [/host](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/host/) contains Browser process.  
The communication between GPU and Browser process is done via mojo API.  
Mojo API is defined in [/mojom](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/mojom/).  

### What are thier responsibilities?
Browser process is responsible for a general stuff.  
It configures the window settings such as window state, bounds, focus activation and so on.  
It's also responsible for communicating with server via wayland protocol.  
Browser process creates `wl_buffer` object which represents a buffer to draw a content.

On the other hand, GPU process does "Graphic"-ish stuff.  
GPU process is good at parallel computing with many computation units.  
GPU process is responsible for drawing/painting content.  
"Drawring" is producing a frame and setting it into the buffer created by Browser process.  
The drawn content is sent to Browser Process.

When the whole opertion went OK, the buffer is marked as ready via [OnSubmission](https://source.chromium.org/chromium/chromium/src/+/main:ui/ozone/platform/wayland/gpu/wayland_surface_gpu.h;l=30;drc=3e1a26c44c024d97dc9a4c09bbc6a2365398ca2c).

## Inter-process communication for ChromeOS
Browser process is a wayland client and communicates with both wayland Server and GPU process.  
Note that GPU process does not interact with wayland Server.  
This is the same for Server side, Browser process in Server communicates with the client, not GPU process.  

So the flow would be like:  
Action  
-> Client Browser: handle something and requests drawing  
-> Client GPU: drawing content  
-> Client Browser: Recive the content  
-> via wayland protocol  
-> Server Browser: handle something and requests drawing with given content  
-> Server Gpu: drawing content together with other applications  
-> Server Browser: Receive the content  
(-> via wayland protocol if server needs to configure)  
...  