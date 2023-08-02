# Exo Window Coordinates

Window is located on specified coordinate on display.  
Each window has bounds like (42,173 773x626) constructed as [gfx::Rect](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/geometry/rect.h;l=37;drc=3e1a26c44c024d97dc9a4c09bbc6a2365398ca2c).  
Handling coordinates has been a very difficult part on compositing.  
This note explains what is going on inside Exo about Bounds.

## Surface Role and Bounds
Bounds are set as follows.

![](https://hackmd.io/_uploads/Sk7jScgDh.png)

Host window owns one single [RootSurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface_tree_host.h;l=186;drc=d9095892389e9b1be3b9068feb1330d87f35c61e).
RootSurface represents toplevel window.  
Host window ([ExoShellSurfaceHost](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface_tree_host.h)) bounds is decided by [whole surface hierarchy](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.h;l=635;drc=d9095892389e9b1be3b9068feb1330d87f35c61e). This is calculated in [CommitSurfaceHierarchy](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/surface.cc;l=927;drc=d9095892389e9b1be3b9068feb1330d87f35c61e).  
Host window bounds covers all subsurfaces in the hierarchy as the figure above.

Host window's parent is ExoShellSurface which controls window state and window bounds. Here, window bounds is visual bounds that the users see as a window. In this image, it is the same as root surface bounds.

## How bounds are stored
Bounds are set as relative coordinate against the parent window, not the screen coordinate.

ExoShellSurface is the same to root surface and its parent is `Desk_Container_A` which represents the display.

Host window coordinate is negative against ExoShellSurface in the figure above. This is translated at [UpdateHostWindowBounds](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/surface_tree_host.cc;l=381;drc=1dc2baac30e93445801952ca203080c7a5bd9f04).
Window hierarchy bounds looks like this:
```
[window] ExoShellSurface-0 bounds=0,149 716x575
    [window] ExoShellSurfaceHost-0 bounds=-1,-188 718x764
        [window] ExoSurface-0 bounds=1,188 716x575
            [window] ExoSurface-3 bounds=0,509 716x66
            ...
            [window] ExoSurface-10 bounds=38,-188 640x360
```
In this example, `ExoSurface-0` is RootSurface, and `ExoSurface-10` is a subsurface outside of RootSurface.  
Since `ExoSurface-10` is above by 188px, ExoShellSurfaceHost a.k.a host window coordinate becomes -188px against ExoShllSurface.  

Also, note that other ExoSurface coordinates are relative to RootSurface `ExoSurface-0`.

### On Client Side
On client side, the window bounds is naturally stored accordingly to toplevel window's origin.  
This mismatch must be handled accurately between Server and Client.

## Why Host Window is Surface Hierarchy
It looks easier to handle if host window is same as root surface.  
The reason of letting host window covers whole subsurface hierarchy is for "Clipping".

### Clipping
Each windows contain several frames (quads). They are composited to become one window.  
On compositing window, we may need to clip the frame partially for example when the frame is scrolled out from the toplevel window.

![](https://hackmd.io/_uploads/B1z-AOlD3.png)

For example, YouTube video area is handled as one quad. When you scroll down the YouTube site, video is partially out-side of the window. In such scenario, we still receive the full frame of YouTube and clip the topside of the frame, so that we won't see YouTube video going out of the window area.

### Frames outside of RootSurface
On the other hand, there is a case where we want keep frames unclipped by RootSurface bounds, for example popup bubble view rectangle.  
In such case, by covering whole subsurfaces by host window, we can theoritically keep surfaces on viz.
