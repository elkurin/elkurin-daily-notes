# Coordinates Conversion in Layer Tree 

Coordinates conversion is always a hard problem.  
Today, let's look into how the conversion works inside UI layers.

## ConvertPointToLayer
[ConvertPointToLayer](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer.cc;l=833;drc=3e8ca029a67e50cd5a73ae784da40b05c63df153) converts a `point` from `source` layer to `target` layer.

```cpp=
void Layer::ConvertPointToLayer(const Layer* source,
                                const Layer* target,
                                bool use_target_transform,
                                gfx::PointF* point) {
  if (source == target)
    return;

  const Layer* root_layer = GetRoot(source);
  CHECK_EQ(root_layer, GetRoot(target));

  if (source != root_layer)
    source->ConvertPointForAncestor(root_layer, use_target_transform, point);
  if (target != root_layer)
    target->ConvertPointFromAncestor(root_layer, use_target_transform, point);
}
```

If `source` is same to `target`, it immediately returns `point` as is.  
If not, we obtain `root_layer` from [GetRoot](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer.cc;l=60;drc=6f4c64436342c818aa41e6a5c55034e74ec9c6b6).  
GetRoot finds the top-most layer by accessing `layer->parent()` until it becomes null.  

Then it convert `point` from `source` into `root_layer` and then `root_layer` to `target`.
`source` -> `root_layer` is done by [ConvertPointForAncestor](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer.cc;l=1495;drc=6f4c64436342c818aa41e6a5c55034e74ec9c6b6).  
`root_layar` -> `target` is done by [ConvertPointFromAncestor](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer.cc;l=1506;drc=6f4c64436342c818aa41e6a5c55034e74ec9c6b6).  

They both gets transform from [GetTransformRelativeToImpl](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer.cc;l=1870;drc=6f4c64436342c818aa41e6a5c55034e74ec9c6b6).  
Supposed the layer tree is from 1~10 and each transform is translate1, scale1,...  
The calculation will be:  
`* scale1 -> + translate1 -> * scale2 -> + translate2 -> * scale 3 ...`

## How transform is applied
[gfx::Transform](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/geometry/transform.h;l=45;drc=6f4c64436342c818aa41e6a5c55034e74ec9c6b6) represents a matrix to scale and offset to the target coordinates.  
Here, we focus on [AxisTransform2d](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/geometry/axis_transform2d.h;l=28;drc=6f4c64436342c818aa41e6a5c55034e74ec9c6b6).

It has two parameters, scale and translation.  
[scale_](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/geometry/axis_transform2d.h;l=143;drc=6f4c64436342c818aa41e6a5c55034e74ec9c6b6) represents how much we multiply to adjust to the coordinates.  
[translation_](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/geometry/axis_transform2d.h;l=144;drc=6f4c64436342c818aa41e6a5c55034e74ec9c6b6) represents how much we should add.

Generally, [PostConcat](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/geometry/axis_transform2d.h;l=62;drc=6f4c64436342c818aa41e6a5c55034e74ec9c6b6) is used.  
This logic calculates scale first and then translate.
```cpp=
void PostConcat(const AxisTransform2d& post) {
  PostScale(post.scale_);
  PostTranslate(post.translation_);
}
```

### Matrix44
We only discussed about AxisTransfom2d above, but actually the transform container can take two types:

```cpp=
  bool full_matrix_ = false;
  union {
    AxisTransform2d axis_2d_;
    Matrix44 matrix_;
  };
```
It's implemented in union, so one of them is used.  
`full_matrix_` is a flag to check which of AxisTransform2d and Matrix44 is used.

[Matrix44](https://source.chromium.org/chromium/chromium/src/+/main:ui/gfx/geometry/matrix44.h;l=36;drc=6f4c64436342c818aa41e6a5c55034e74ec9c6b6) is also the underlying data structure of Transform.  
While AxisTransform2d is a subset of 2# linear transforms that only translation and uniform scaling are allowed, this is a clear 4x4 matrix.
```cpp=
   \  col
r   \     0        1        2        3
o  0 | scale_x  skew_xy  skew_xz  trans_x |   | x |
w  1 | skew_yx  scale_y  skew_yz  trans_y | * | y |
   2 | skew_zx  skew_zy  scale_z  trans_z |   | z |
   3 | persp_x  persp_y  persp_z  persp_w |   | w |
```
This class is more general, but we use AxisTransform2d for most cases (if we can uses it) since it's easier to understand.

## Use case
For Exo surfaces, the hierarchy is like this:
```
Display
-> ExoShellSurface
   -> Host window
      -> Root surface
         -> Sub surface
```
[Window::ConvertPointToTarget](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window.cc;l=594;drc=6f4c64436342c818aa41e6a5c55034e74ec9c6b6) calls [Layer::ConvertPointToLayer](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer.cc;l=833;drc=3e8ca029a67e50cd5a73ae784da40b05c63df153) and goes through all the way around.

Display, ExoShellSurface and Host window is in DP while Root surface and Sub surface is in pixel.  
ExoShellSurface is located somewhere inside display and it's very likely that the offset from display to ExoShellSurface is non-zero.  
Host window and root surface may also have offsets.

Let's see between root surface and display.

In this use case, pixel -> dp scale is set as transform between root surface -> host window.  
It is `1.f / device_scale_factor`.  
Other than that, it adjust the translation offsets in dp.  
Therefore, the value added up is something like:
`1.f / device_scale_factor` as `scale_`, and sum of dp offsets as `translation_`.

## Note
Do we really need to get `root_layer` to compare the scale?  
We can just obtain the nearest common parent?
