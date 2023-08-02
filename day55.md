# Pixel and dip coordinates

ChromeOS compositing cannot be done without coordinates discussion.  
Here, let's take a look on pixel and dip coordinates usage between Exo and Wayland.  

## Pixel vs DIP
Pixel is a minimum addressable element on screen. Its size depends on the devices, so the actual size becomes smaller on dense pixel devices and vise versa.  
On the other hand, DIP stands for **D**ensity **I**ndependent **P**ixel. It specifies the actual position and size.  

We mainly use pixel coordinates on compositing. The position in pixel is always integer.  
For Lacros, we send position in pixel coordinates by default. [kWaylndSurfaceSubmissionInPixelCoordinates](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/common/features.cc;l=20-23;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687) flags is `base::FEATURE_ENABLED_BY_DEFAULT` for Lacros platform. If it's disabled, all states are converted into DIP by scaling with 1.f / [GetWaylandScale()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_surface.cc;l=372;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687).  

We sometimes want to scale the rendered contents into other size, for example, expands the content for users with poor eyesight.  
This scaling on server side.  
We translate the pixel coordinates position into DIP coordinates if needed. 
When device scale factor is set to non-one, we scale the position accordingly to the given scale.  
While the position in pixel is always integer, the position in DIP is float since the device scale factor can take float number.  

The conversion from pixel to dip is like this:
```cpp=
PointF ConvertPointToDips(const Point& point_in_pixels,
                          float device_scale_factor) {
  return ScalePoint(PointF(point_in_pixels), 1.f / device_scale_factor);
}
```
It is scaled by `device_scale_factor` specified by device, via settings app for ChromeOS.  
As you can see, `point_in_pixels` is `Point`, the integer point, while returned dip point is in `PointF`, the float point.  

For example, we tranlate the host window size into DIP in [UpdateHostWindowBounds](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/surface_tree_host.cc;l=408;drc=6790999f98e4bcf6242dc7894cd44ebbe53982ac).  


## Subpixel
In Chrome, renderer translates the web contents into pixels. The output is on pixel coordinates. After creating the web content, we sometimes scale the content. In such cases, the bounds may become float number while but still we need to adjust the content into integer since pixel is a smallest element in screen.  
On adjusting the content bounds, we try to make the image sharp by rounding the bounds to the approprite integer.

Such adjustment is done by [SetSubpixelPositionOffset](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer.h;l=217;drc=ce83030f1b647f5a6d2986a3fe55d45ee0468ad0). If there is no offset specified, [Layer::RecomputePosition](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/layer.cc;l=1777;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687) will just use `bounds_origin()` as the position while `GetSubpixelOffset()` is added for adjustment if it's specified.  

Adjustment bounds is sent from Views.  
On [View::SetBounds](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/view.cc;l=376;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687), [`offset_data`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/view.cc;l=403-406;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687) is passed to [SetLayerBounds](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/view.cc;l=3209;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687)->[SnapLyerToPixelBoundary](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/view.cc;l=3118;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687)->[SetSubpixelPositionOffset](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer.h;l=217;drc=ce83030f1b647f5a6d2986a3fe55d45ee0468ad0).  
This offset data is handled as [LayerOffsetData](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/view.h;l=338;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687).  
Offset to set is calculated by `LayerOffsetData() + bounds_.OffsetFromOrigin()`.  
Looking at [LayerOffsetData operator+](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/view.h;l=361;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687), it's inner impl is here:  
```cpp=
void AddOffset(const gfx::Vector2d& offset_to_parent) {
  offset_ += offset_to_parent;

  gfx::Vector2dF fractional_pixel_offset(
      offset_to_parent.x() * device_scale_factor_,
      offset_to_parent.y() * device_scale_factor_);

  gfx::Vector2d integral_pixel_offset =
      gfx::ToRoundedVector2d(fractional_pixel_offset);

  rounded_pixel_offset_ += integral_pixel_offset - fractional_pixel_offset;
}
```
It's rounding a fractional pixel offset scaled by `device_scale_factor_` to integer.  
[gfx::ToRoundedVector2d](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/geometry/vector2d_conversions.cc;l=20;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687) calls [ClamRound](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/numerics/safe_conversions.h;l=380;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687) for `x` and `y`. This rounding logic is:  
```cpp=
Dst ClampRound(Src value) {
  const Src rounded =
      (value >= 0.0f) ? std::floor(value + 0.5f) : std::ceil(value - 0.5f);
  return saturated_cast<Dst>(rounded);
}
```
It's 四捨五入.  

Considering this behavior, when we gradually move the surface, there is a point where the position instantly changes, instead of smoothly moved.  


Inside ui::compositor, [Layer::SubpixelPositionOffsetCache](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/layer.cc;l=122;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687) manages the subpixel offset data.  
The subpixel offset passed from views is cached to [`subpixel_position_offset_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/layer.h;l=728;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687) via [SetExplicitSubpixelPositionOffset](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/layer.cc;l=172;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687).  

### Can we set subpixel position from Exo?
The above path is coming from Views.  
How aboud Exo setting position offset? I think it's possible.  
This usage would be considerble when Exo sends the pixel coordinates and Exo translates it into dip.
