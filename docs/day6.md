# Recreating Layer

## Concept of Window and Layer
To understand the rough picture, let's take a look at real use case of layer.
On maximizing the window, there is an animation smoothly transferring to the smaller size to the maximized size. This is called CrossFadeAnimation. [impl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/wm/window_animations.cc;l=253;drc=c4efcabd32a851d4d7816206d3c8793a55e9d57e)

It runs really fast, so it's a bit difficult to observe, but if you carefully look at what happens on maximizing window, you can see there are two layers overlaying others.  
What is happening is like this:
![](https://hackmd.io/_uploads/HkEuaY4I3.png)

During the animation, the old layer is gradually transformed from its original position and size to the new position and size. The old layer is also faded out, so that it becomes less opaque over time.  
At the same time, the new layer is gradually transformed from its original position and size to the new position and size. The new layer is also faded in, so that it becomes more opaque over time.


But these both layers represent one same window. Google Search tab.

This visual frame is "**Layer**" and the feature is "**Window**".

## What is ui::Layer
[ui::Layer](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer.h) is a similar concept to [aura::Window](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window.h).  
Each [aura::Window](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/aura/window.h) owns up to one layer (usually exactly one), but a layer can be detached to attach another layer.  
Layer manages a texture and transform, so Layer decides what the window looks like. If an attached layer is taken away from a window, the window is freezed.  
Layer is managed as a tree. The root layer holds compositor.

[ui::Layer](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer.h) has [`surface_layer_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/layer.h;l=794;drc=c4efcabd32a851d4d7816206d3c8793a55e9d57e) who is [cc::SurfaceLayer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/surface_layer.h). SurfaceLayer renders a surface of the compositor output.

Generally, layer is owned by [ui::LayerOwner](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/layer_owner.h). Note that aura::Window is overriding LayerOwner.

## Detaching Layer
[LayerOwner::AccuireLayer()](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer_owner.h;l=48;drc=3e1a26c44c024d97dc9a4c09bbc6a2365398ca2c) allows us to detach layer from the current layer owner and get the layer ownership.  
`layer_owner_` is moved to the caller, but `layer_` continues to have an access to the layer.

This is used when we want to animate the presentation of the owner just prior to destroying it.  
For example, on destroying the window, the window does not instantly disappear but instead fade out. On maximizing the window, it smoothly transitions to the maximized size (not smooth on Linux though...).

## Recreating Layer
[LayerOwner::RecreateLayer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/layer_owner.cc;l=53;drc=c4efcabd32a851d4d7816206d3c8793a55e9d57e) does the followings:
1. [AcquireLayer](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer_owner.h;l=48;drc=3e1a26c44c024d97dc9a4c09bbc6a2365398ca2c) as an `old_layer`.
2. Clone `old_layer` and [SetLayer](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer_owner.h;l=70;drc=3e1a26c44c024d97dc9a4c09bbc6a2365398ca2c) to LayerOwner as a new layer.
3. If `old_layer` has a parent, insert new layer as a sibling stacked below `old_layer`. If not, it implies `old_layer` is the layer tree root, so move the Compositor move to the new layer.
4. Migrate all child layers of `old_layer` to new layer.

At this point, there are two layers representing same window like what we saw on maximizing animation on Concept section.

## Layer::SetShowSurface
[Layer::SetShowSurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/layer.cc;l=1058;drc=c4efcabd32a851d4d7816206d3c8793a55e9d57e) begins showing content from a surface with surface_id. This is called inside [Clone](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/layer.cc;l=277-283;drc=c4efcabd32a851d4d7816206d3c8793a55e9d57e).  
It does clone all params of the current layer including surface_id.   
TODO(elkurin): Investigate surface_id expectation here.