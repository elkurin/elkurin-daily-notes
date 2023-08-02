# Clipping in cc

This notes dig into the clip rect flow inside cc.

## What is clip rect
CompositorFrame is produced accordingly to the requested size. After that, we may want to clip it to the size on display.  
For example, think about scrolling the window. It's costful to rerender whole frames on each scroll events, instead we can use the same frame larger than the size fit into the screen by selecting the different area corresponding to the scrolled area.  
It could be useful if we can clip the content by window bounds, or other specified rectangle.  

We use this clipping mechanism for many places. Let's see how it is handled inside cc.

## Background: PropertyTree
We holds the data as a tree strucure for each property. [PropertyTree](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/property_tree.h;l=75;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) is a base class of tree and there are classes inheritting PropertyTree for each property.  
[ClipTree](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/property_tree.h;l=339;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) is one of them. This is for clip rect property.  
[PropertyTree](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/property_tree.h;l=75;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) provides a method to access to the node such as [`Node(int i)`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/property_tree.h;l=89;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188),   
[`parent(T* t)`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/property_tree.h;l=98;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188), [`Insert(T& tree_node, intparent_id)`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/property_tree.h;l=87;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) and so on. The node is specified by template parameter `T`. As for ClipTree, [ClipNode](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/clip_node.h;l=30;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) is specifid. It holds `clip` value which represents the clip rect that this node contributes, and other ids.  

## How clip rect is set to DrawProperties
 [`CalculateDrawProperties`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/draw_property_utils.cc;l=1563;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) is an API to calculate draw properties including clip rect and this will update the parameters of the corredponding [ClipNode](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/clip_node.h;l=30;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) via [ClipTree](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/property_tree.h;l=339;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) interface.  
This API is used from cc::Scheduler when [ProcessScheduledActions](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/scheduler/scheduler.cc;l=852;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) for draw actions (DRAW_IF_POSSIBLE or DRAW_FORCED). To make it simple, it's used when cc starts drawing.  

Inside [`CalculateDrawProperties`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/draw_property_utils.cc;l=1563;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188), [SetSurfaceClipRect](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/draw_property_utils.cc;l=608;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) will extract the rect registered to [ClipTree](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/property_tree.h;l=339;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) into [RenderSurfaceImpl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/render_surface_impl.cc).   

What we want to know next is how the clip rect is specified on ClipTree.  
The answer is from [Layer::SetClipRect](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer.cc;l=587;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188).  
Clip rect calculation is like this:
```cpp=
gfx::RectF effective_clip_rect = EffectiveClipRect();
if (ClipNode* node =
    property_trees->clip_tree_mutable().Node(clip_tree_index())) {
  node->clip = effective_clip_rect;
  node->clip += offset_to_transform_parent();
  property_trees->clip_tree_mutable().set_needs_update(true);
}
```
[EffectiveClipRect](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer.cc;l=622;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) returns the clip rect which is inside the bounds of the target (since only the area inside bounds is effective).  
The clip we refer to is clip rect inside [LayerTreeInputs](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer.h;l=1028;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188). This `clip_rect` is relative to this layer.  

Another way we have to update clip rect values is from [LayerTreeHost::UpdateLayers](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host.cc;l=818;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) called from [BeginMainFrame](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/proxy_main.cc;l=134;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188).   TODO(elkurin): Double-check this. Where does clip rect come from?
Inside it, [ComputeClips](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/draw_property_utils.cc;l=840-846;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) accumulates clip rect using parent clip.  

## How RenderSurfaceImpl handles clip rect
[RenderSurfaceImpl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/render_surface_impl.cc) holds clip rect and other properties in`draw_properties_`. This stored values are referred on [AppendQuads](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/render_surface_impl.cc;l=435;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) to pass to [SharedQuadState](https://source.chromium.org/chromium/chromium/src/+/main:components/viz/common/quads/shared_quad_state.h).  

### Inside PictureLayerImpl
*We take a look on PictureLayer as a representitive of cc layers.*  
clip rect in SharedQuadState is updated to fit in the target space ([code](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/picture_layer_impl.cc;l=267-268;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188)).  
This data is set to RenderPass by [CreateAndAppendDrawQuad](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/quads/compositor_render_pass.h;l=88;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188). From here, it will be handled on viz.  

### Inside DirectRenderer
[DirectRenderer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/direct_renderer.cc) is a renderer in viz. "Direct" implies that it does not delegate rendering to another compositor.  

clip rect is set via [SetScissorStateForQuad](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/direct_renderer.cc;l=505;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188).  
The clip rect is updated to fit in [MoveFromDrawWindowSpace](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/service/display/direct_renderer.cc;l=151;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) which uses `current_draw_rect_` offset.  

## Note
### clip rect on child surface
Here's the current impl of [EffectiveClipRect](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer.cc;l=622;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188):
```cpp=
gfx::RectF Layer::EffectiveClipRect() const {
  const gfx::RectF layer_bounds = gfx::RectF(gfx::SizeF(bounds()));
  if (clip_rect().IsEmpty())
    return layer_bounds;

  const gfx::RectF clip_rect_f(clip_rect());

  if (masks_to_bounds() || mask_layer() ||
      filters().HasFilterThatMovesPixels())
    return gfx::IntersectRects(layer_bounds, clip_rect_f);

  return clip_rect_f;
}
```
It looks like this EffectiveClipRect is the one who is limiting the bounds only inside layer bounds while clip rect may be outside of the layer?  
TODO(elkurin): Check if it's correct. Attension to the parent layer bounds.  

### gfx::Rect::Intersect
[Rect::Intersect](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/gfx/geometry/rect.cc;l=162;drc=1b9ee37d9e583adb8b4f492115bdb8fd268e8188) updates the rect to fit in the given gfx::Rect reference `rect`.  
It sets the intersection as the name says. If there is no area, it becomes (0, 0, 0, 0).
