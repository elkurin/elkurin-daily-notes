# Resource Pool

In the [Frame Eviction](/p7Xlv09bTgKBczGScYAnJA) note, we discussed how to discard resources wisely. Such action is tracked by [ResourcePool](https://source.chromium.org/chromium/chromium/src/+/main:cc/resources/resource_pool.h) which oversees resources usage from both renderer and viz.  
This note aims to understand the ResourcePool's responsibility and how it works by reading codes.  

## Overview
[ResourcePool](https://source.chromium.org/chromium/chromium/src/+/main:cc/resources/resource_pool.h) is a resource tracker in cc (see [Graphic Pipeline on Chrome Compositor (cc)](/5ikmEAt9TVGbg1NrYA1EJw)).  
It keeps track of which resources (such as tiles) are used by the renderer nd which are held by viz.

For example, [TileManager::CreateRasterTask](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/tiles/tile_manager.cc;l=1256;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) aquires a resource from [AcquireResource](https://source.chromium.org/chromium/chromium/src/+/main:cc/resources/resource_pool.cc;l=184;drc=8529cb55df3c89cace2cf3f828314d46a030bcad) and this resource is being used to [AcquireBufferForRaster](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/raster/raster_buffer_provider.h;l=55;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936).  

## Implementation
[ResourcePool](https://source.chromium.org/chromium/chromium/src/+/main:cc/resources/resource_pool.h) hold three types of resource storage, [`unused_resources_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=482;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) and [`busy_resources_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=483;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) as circular_deque and [`in_use_resources_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=486;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) as std::map.  

Each resource is held as [PoolResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=298;drc=edb62228662be757def4d8bcbc49bc5b10118bd8).  
Its current state is stored in [`state_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=428;drc=edb62228662be757def4d8bcbc49bc5b10118bd8).  
* State::kUnused: Resource is free for reusing/releasing.
* State::kInUse: Resource is being used by viz as [InUsePoolResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=126;drc=edb62228662be757def4d8bcbc49bc5b10118bd8).  
* State::kBusy: Resource has been exported to viz process.  

When you want to get a resource, call [AcquireResource](https://source.chromium.org/chromium/chromium/src/+/main:cc/resources/resource_pool.cc;l=184;drc=8529cb55df3c89cace2cf3f828314d46a030bcad). First, it attemps to reuse resources in [`unused_resources_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=482;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) queue by calling [ReuseResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=131;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936). If there is a resource whose format, size and color space meets requirement for the resource we are trying to aqcuire, we move it to [`in_use_resources_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=486;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) and send back as a return value of [ReuseResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=131;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936).  
If there is no resource to reuse, [CreateResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=164;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936). At this point, resource is marked as ['State::kInUse']  
After getting PoolResource by reusing or creating, it will be wrapped as [InUsePoolResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=126;drc=edb62228662be757def4d8bcbc49bc5b10118bd8) to return.  

[InUsePoolResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=126;drc=edb62228662be757def4d8bcbc49bc5b10118bd8) is a scoped move-only object. This is used to get a resource from the pool and takes [PoolResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=298;drc=edb62228662be757def4d8bcbc49bc5b10118bd8) reference as a pointer. This does not use raw_ptr<..> for performance reason.  
On destructing, make sure `resource_` is still alive so that [InUsePoolResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=126;drc=edb62228662be757def4d8bcbc49bc5b10118bd8) can return [PoolResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=298;drc=edb62228662be757def4d8bcbc49bc5b10118bd8) ownership to the pool.  

[InUsePoolResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=126;drc=edb62228662be757def4d8bcbc49bc5b10118bd8) cane be release by [ReleaseResource](https://source.chromium.org/chromium/chromium/src/+/main:cc/resources/resource_pool.cc;l=376;drc=8529cb55df3c89cace2cf3f828314d46a030bcad).  
On releasing, ResourcePool will transfer resource to `unused_resources_` or `busy_resources_` or delete the resource depending on the state.  
If resource contains valid [viz::ResourceId](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/resources/resource_id.h;l=23;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936), transfer to `busy_resources_` since it implies the resource is passed to viz and under composition. If not, you usually give it back to `unused_resources_` from [DidFinishUsingResource](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=508;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) except for the case [when avoid_reuse is set to true](https://source.chromium.org/chromium/chromium/src/+/main:cc/resources/resource_pool.cc;l=423;drc=8529cb55df3c89cace2cf3f828314d46a030bcad) which deletes PoolResource by [DeleteResouece](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=486;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936).  
[`avoid_reuse_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=409;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) is marked as false by default, but this could be overridden by [InvalidateResources](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=361;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) from [LayerTreeHostImpl::CleanUpTileManagerResources](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host_impl.cc;l=3737;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) to avoid reusing previously allocated resources, or from [PrepareForExport](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=325;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) on failing to allocate GpuMemoryBuffer.  

## Memory saving
[EvictExpiredResources](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=528;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) will evict the resources accordingly to the last used timestamp.  
Each PoolResource has [`last_usage_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.h;l=404;drc=3ecc0c281a06f6cb6d7856aac560e45feb554f53) timestamp. [EvictResourcesNoUsedSince](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=553;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) checks the timestamps for all resources queued in `unused_resources` and [DeleteResouece](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=486;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) if they are not used for more than `time_limit`.  

Also, there is an API to reduce resource usage called [ReduceResourceUsage](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=459;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936).  
This reduction is not automtically, but instead you need to explicitly call it to reduce the resources. For example, this is called from [TileManager::DidFinishRunningAllTileTasks](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/tiles/tile_manager.cc;l=585;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936) to delete all unused resources after completing all tasks.  

The maximum resource that is allowed to store is kept as `max_memory_usage_bytes_` and `max_resource_count_`. There is a limitation on both totl bytes and the number of resources. These limitation values are set from [SetResourceUsageLimits](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/resources/resource_pool.cc;l=451;drc=bc1deb5c0b4cdbc7a83398e8fc8e09d00edbc936).  
By defult, it set as 0 which implies there should be no unused memory kept after redusing resource.  

To summarize, there are two ways to clean up unused resources:
1. By timestamp (automatically)
2. By maximum bytes/number of resources (manually)