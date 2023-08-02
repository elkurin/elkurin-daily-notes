# Shadow and gfx library

Around the chrome window, there is a decoration called "shadow".  
"Shadow" can be observed on the edge of the window as a black blur area.  

How is it implemented?

## Shadow bounds and owner
[ui::Shadow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor_extra/shadow.cc) is its implementation.  

`content_bounds_` shows the bounds of the window and shadow bounds is also calculated from `content_bounds_`.  
`content_bounds_` is set from [Shadow::SetContentBounds](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor_extra/shadow.cc;l=34;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) and for Lacros window, it is called from [ShellSurfaceBase::UpdateShadow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/shell_surface_base.cc;l=1754;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) when [CommitWidget](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/shell_surface_base.cc;l=1960;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) or [OnWindowBoundsChanged](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/shell_surface.cc;l=456;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061).  

On `content_bounds_` is set, [UpdateLayerBounds](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor_extra/shadow.cc;l=134;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) is called and calculte the shadow bounds.  
Shadow margings are negative as it's around the window, so we need to expands outwards from `content_bounds_`.

Shadow is [ui::Layer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/layer.h), not the window.  
It is owned by [`shadow_layer_owner_`](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor_extra/shadow.h;l=111;drc=89e1f63c97ae8c66ff1b8de4fbf564978349ac70) and handled from [SystemShadowOnNinePatchLayer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/style/system_shadow_on_nine_patch_layer.cc) which is responsible for painting shadow.  
TODO(elkurin): Wht is nine patch layer?

## gfx class and functions
Inside [UpdateLayerBounds](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor_extra/shadow.cc;l=134;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061), there are several gfx libraries.  

### [gfx::Insets](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/geometry/insets.h)
The inner implementation is in [InsetsOutsetsBase](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/geometry/insets_outsets_base.h). This represents the widths of the four borders or margings.  
The member variables are `top_`, `left_`, `bottom_` and `right_` who are int type. As you can see, we can defined each width independently.

[gfx::Insets](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/geometry/insets.h) can represent a space within a rectangle by shrinking the rectble by the inset amount on all four sides. Alternatively it can represent a border that has a different thickness on each side.  
As for shadow, the latter usage is applied.  

It's used like this:
```cpp=
const gfx::Insets margins = gfx::ShadowValue::GetMargin(details.values);
gfx::Rect new_layer_bounds = content_bounds_;
new_layer_bounds.Inset(margins);
x libgfx::Rect shadow_layer_bounds(new_layer_bounds.size());
```

### [gfx::ShadowValue](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/shadow_value.h;l=26;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061)
[gfx::ShadowValue](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/shadow_value.h;l=26;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) encapsulates parmeters needed to define a shadow.  
Member variables are `offset_`, `blur_` and `color_`.
`offset_` is just an offset of the bounds.  
`blur_` is double type in pixel. It represents the width of the shadow blur. The value is `kBlurCorrection * elevation * 2` or `kBlurCorrection * elevation`.   
`color_` is [SkColor](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/skia/include/core/SkColor.h;l=37;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061). By default it uses [`SK_ColorTRANSPARENT`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/skia/include/core/SkColor.h;l=99;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061), but as for MD shadows and ChromeOS system ui, [`SK_ColorBLACK`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/shadow_value.h;l=69;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) is used.

### [gfx::ShadowDetails](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/shadow_util.h;l=27;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061)

As for [ui::Shadow](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor_extra/shadow.cc), ShadowValues are calculted by [MakeMdShdowValues](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/shadow_value.cc;l=109;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) by default and  [MakeChromeOSSystemUiShadowValues](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/shadow_value.cc;l=130;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) for some features in ChromeOS platform.  
This ShadowValues is included by [gfx::ShadowDetails](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/shadow_util.h;l=27;drc=1890f3f74c8100eb1a3e945d34d6fd576d2a9061) as a member.
