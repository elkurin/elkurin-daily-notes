# Surface Origin during Resizing

Surface origin specifies the position of a surface.  
The origin is a top-left position.  
This value may change during resizing the window.  
Let's look at how origin is calculated during resizing.

## GetSurfaceOrigin
[GetSurfaceOrigin](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/shell_surface.cc;l=419;drc=48290eaa6a80203b317bff0b5e6cf91005602c41) is used to calculate the origin as the name says.  
This origin is on parent coordinates. Relative to its parent.  
In this case, it's on ExoShellSurfaceHost.

Here, `visible_bounds` is a geometry. It wraps the whole visible area for this window.  
`client_bounds` is also a bounds for the window, but this does not include NonClientFrameView such as a top bar of ARC window.  
[`origin_offset`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/shell_surface.h;l=240;drc=48290eaa6a80203b317bff0b5e6cf91005602c41) is a offset from ExoShellSurfaceHost. This offset may be non-zero only during the resizing since resizing is an asynchronous operation so that ExoShellSurfaceHost may not yet be moved while the surface is already moved.  
`resize_component_` stands for a hit testing mode whose value is one of [HitTestCompat](https://source.chromium.org/chromium/chromium/src/+/main:ui/base/hit_test.h;l=53;drc=3e1a26c44c024d97dc9a4c09bbc6a2365398ca2c) enums. Let's see how this is used in the next section.  
Using these 4 values, the origin is calculated.

fyi: Client, NonClient view description  
![](https://hackmd.io/_uploads/BJ6iHSg_2.png)  

## HitTestCompat
In [GetSurfaceOrigin](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/shell_surface.cc;l=419;drc=48290eaa6a80203b317bff0b5e6cf91005602c41), the operation differs depending on `resize_component_`.  
For all cases, the base position is `- visible_bounds` as the surface origin should be the geometry; the visible area.

* As for HTCAPTION and HTCLIENT, the origin is ajusted by `origin_offset_`. This means the origin is on resizing position.
* As for HTBOTTOM, HTRIGHT and HTBOTTOMRIGHT, the origin is not adjusted but stays as the base position.
* As for HTTOP and HTTOPRIGHT, the origin is adjusted by `client_bounds` only for the x axis.
* As for HTLEFT and HTBOTTOMLEFT, the origin is adjusted by `client_bounds` only for the y axis.
* As for HTTOPLEFT, the origin is adjusted by `client_bounds` for both x and y axis.

As you can see from this, `resize_component_` implies which parts are being resized.  
When you hover the cursor on the edge of window, you can see the cursor form has changed from the pointer to the arrow.  
For example like this:
![](https://hackmd.io/_uploads/BymQFcJ63.png) ![](https://hackmd.io/_uploads/HJMnKckp3.png)

These png can be found in [ui/resources/default_100_percent](https://source.chromium.org/chromium/chromium/src/+/main:ui/resources/default_100_percent/).

`resize_component_` represents which edge is being dragged.  
The default value is HTCAPTION.


## What is Hit Testing?
On the above section, we refered to HitTestCompat value.  
By the way what is hit testing?  

Hit testing is a concept in graphics computing to determin whether a cursor intersects a given graphical object.  
This is used to find where the cursor is placed on so that we can identify the event target.  
What hit testing does is check whether cursor's position is inside the area or not by coordinate calculation.


