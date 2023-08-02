# Draw Properties in cc

On drawing image in cc, it requires properties such as a rectanglar area of the content or the scale value to adjust with the actual display coordinates.  
Let's take a look on properties used for drawring.

## Overview
We can find such properties in [cc::DrawProperties](https://source.chromium.org/chromium/chromium/src/+/main:cc/layers/draw_properties.h).  
It's a container for preperties that layer need to compute before they can be drawn.

## Types in this context
[Transform](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/geometry/transform.h;l=45;drc=7d2d8ccab2b68fbbfc5e1611d45bd4ecf87090b8) is 4x4 matrix and it represents how the given bounds should be translated into the target coordinates. It contains values to scale (x and y) and values to move (x and y). You can construct Transform by [Transform(float scale_x, float scale_y, float trans_x, float trans_y)](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/geometry/transform.h;l=581;drc=7d2d8ccab2b68fbbfc5e1611d45bd4ecf87090b8).  

[Occlusion](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/occlusion.h;l=17;drc=7d2d8ccab2b68fbbfc5e1611d45bd4ecf87090b8) represents the area on the content that is not drawn. This is used to skip unnecessary drawing since the occluded region will not be visible anyway. You can get unoccluded area on content by [Occlusion::GetUnoccludedContentRect](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/occlusion.cc;l=46;drc=7d2d8ccab2b68fbbfc5e1611d45bd4ecf87090b8).  

[Rect](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/geometry/rect.h;l=37;drc=7d2d8ccab2b68fbbfc5e1611d45bd4ecf87090b8) represents a rectangler with x and y as a origin (top-left position) and size and width as a size.  

## Properties
* target_space_transform
Transform from content space to target surface space
* screen_space_transform
Transform from content space to screen space
* occlusion_in_content_space
Occlusion above the layer mapped to the content space
* opacity
Opacity is float number which represents literally "opcacity". If it's 1.f, it is drawn without any transparency. If it's 0.f, it's invisible.
* visible_layer_rect
The bounds of layer which contains all visible contents. In layer coordinates space
* visible_drawable_content_rect
Clipped + visible + drawable content area. In target surface space
* clip_rect
Used to clip the bounds ([Clipping in cc](/LvRsS5aGTNGobStqD0vKcA)). In target surfce space
* mask_filter_info
Stored as [MaskFilterInfo](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/geometry/mask_filter_info.h;l=20;drc=7d2d8ccab2b68fbbfc5e1611d45bd4ecf87090b8). There are two values stored, `rounded_corner_bounds_` which is used to round the edge of the window like the above picture, and `gradient_mask_` which represents a shader based mask stored as [LinearGradient](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/geometry/linear_gradient.h;l=31;drc=7d2d8ccab2b68fbbfc5e1611d45bd4ecf87090b8). (TODO: what's this?)

![](https://hackmd.io/_uploads/ry_6SiA9h.png)


### flags
There are flag booleans:
* screen_space_transform_is_animating
* is_clipped
* is_fast_rounded_corner

## How DrawProperties is used
[LayerImpl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer_impl.h;l=573;drc=7d2d8ccab2b68fbbfc5e1611d45bd4ecf87090b8) stores properties to be computed before drawing.  
This properties are set as quad state via [PopulateShredQuadState](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer_impl.cc;l=147;drc=7d2d8ccab2b68fbbfc5e1611d45bd4ecf87090b8) on [AppendQuads](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/texture_layer_impl.cc;l=110;drc=7d2d8ccab2b68fbbfc5e1611d45bd4ecf87090b8).
