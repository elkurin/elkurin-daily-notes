# cc layers

In cc, there is a concept called "layer".  
What is this?

## What is Layer?
A layer is a 2d rectangle of content with integer bounds.  
It has some transform, clip and effects on it that describe how it should look on screen.

Let's look at [cc::Layer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer.h;l=87;drc=fdfd85f836e0e59c79ed9bf6d527a2b8f7fdeb6e), the base class of all layer.  
[LayerTreeInputs](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer.h;l=1024;drc=fdfd85f836e0e59c79ed9bf6d527a2b8f7fdeb6e) struct is used as a input information in ui compositor.  
It contains for example `clip_rect`, `transform`, `corner_radii`, `scroll_container_bounds`...  
As for clip rect, you can call [cc::Layer::SetClipRect](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer.cc;l=587;drc=fdfd85f836e0e59c79ed9bf6d527a2b8f7fdeb6e) to set it to the layer. This is called, for example, from [ui::Layer::SetClipRectFromAnimation](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/compositor/layer.cc;l=1626;drc=fdfd85f836e0e59c79ed9bf6d527a2b8f7fdeb6e)  

Each layer is an independent unit in the compositor.

## Special Layer and its implementation
All of them inherits [cc::Layer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/layer.h;l=87;drc=fdfd85f836e0e59c79ed9bf6d527a2b8f7fdeb6e).  
Each variant represents the type of layer such as TextureLayer, VideoLayer, SolidColorLayer.  

As described in [Graphic Pipeline on Chrome Compositor (cc)](/5ikmEAt9TVGbg1NrYA1EJw) note, we have Main thread and Compositor thread.  

![](https://hackmd.io/_uploads/SkcM2G2F3.png)
(Cite from [How cc Works](https://source.chromium.org/chromium/chromium/src/+/main:docs/how_cc_works.md))

Each thread has a seperate class that represents the implementation of each layer.  
For Picture Layer, [PictureLayer](https://source.chromium.org/chromium/chromium/src/+/main:cc/layers/picture_layer.h;drc=69e6dc49684309c8b375c4dcd724c6ae61878ecd) for Main Thread and [PictureLayerImpl](https://source.chromium.org/chromium/chromium/src/+/main:cc/layers/picture_layer_impl.h;drc=b2878dbfb8d7c535bd614d5234f73d1617d26469) for Compositor Thread. Note that the work 'impl' does not mean it's an implementation of the abstract class, it's a name of the deperecated thread.

## Special Layers
Let's take a look on what kind of special layers do we have.

### PictureLayer
This layer contains painted content.  
It's responsible for figuring out the target raster scale.  
There are own APIs such as [UpdateRasterSource](https://source.chromium.org/chromium/chromium/src/+/main:cc/layers/picture_layer_impl.h;l=98;drc=b2878dbfb8d7c535bd614d5234f73d1617d26469) and [UpdateTiles](https://source.chromium.org/chromium/chromium/src/+/main:cc/layers/picture_layer_impl.h;l=103;drc=b2878dbfb8d7c535bd614d5234f73d1617d26469).
PicureLayerImpl holds [`tilings_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/layers/picture_layer_impl.h;l=245;drc=fdfd85f836e0e59c79ed9bf6d527a2b8f7fdeb6e) who is [PictureLayerTilingSet](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/tiles/picture_layer_tiling_set.h;l=29;drc=fdfd85f836e0e59c79ed9bf6d527a2b8f7fdeb6e).

### TextureLayer
GPU texture's layer, display gpu or software resources.  
It contains the rendereed output of a plugin instance.

### SurfaceLayer
This layer is a surface rendered referencing the output of another compositor.  
Surface layer hs a surface id to refer some other stream of compositor frames.

### SolidColoLayer
Very straight forward.  
A layer known to be merely a solid color.  
This is seperated so that we can skip spending resources on raster work.  
As you can see in [SolidColorLayer](https://source.chromium.org/chromium/chromium/src/+/main:cc/layers/solid_color_layer.h;drc=2ae4b7c6500f5b0b5480cb2db823fd3d1d80387b) and [SolidColorLayerImpl](https://source.chromium.org/chromium/chromium/src/+/main:cc/layers/solid_color_layer_impl.h;drc=3f7a9d89de4c434583d520384b01ce87178ce888), there is almost nothing.  

### SolidColorScrollbarLayer
Similar concept to SolidColorLayer, but for scrollbar.  
This can be fully drawn on Main thread.

## Note
Inside cc, we have [ProtectedSequenceSynchronizer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/base/protected_sequence_synchronizer.h) which can be used to enforce thread safety.  

For example, [ProtectedSequenceReadble](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/base/protected_sequence_synchronizer.h;l=76;drc=fdfd85f836e0e59c79ed9bf6d527a2b8f7fdeb6e) values are
- readable by the owner thread without blocking
- writable by the owner thread but not during a protected sequence
- readable by a non-owner thread during protected sequence
- never writeble by a non-owner thread

We have many this annotation in cc, for example `ProtectedSequenceReadable<bool> stretch_content_to_fill_bounds_;` in SurfaceLayer.
