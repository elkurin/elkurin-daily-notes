# SurfaceId and synchronization

In Chromium compositing, Multiple clients such as Browser UI and Blink renderer submit CompositorFrame and viz composites them into once frame.  
These communication is done asynchronously so we need to make sure the surface is synchronized between viz clients.  
SurfaceId is a fundamental component of the surface synchronization.  
Let's look into SurfaceId.

## SurfaceId
SurfaceId consists of two components:
* [FrameSinkId](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/surfaces/frame_sink_id.h;l=38;drc=0c3a07090f8fc813207632bbf6c3196b9d7e5cf1)
* [LocalSurfaceId](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/surfaces/local_surface_id.h;l=66;drc=0c3a07090f8fc813207632bbf6c3196b9d7e5cf1)

FrameSinkId uniquely identifies a CompositorFrameSink and a viz client.  
LocalSurfaceId uniquely identifies a sequentially alocated ID of CompositorFrames in a particular client.  
The set of these two IDs uniquely identifies a surface globally across all clients.

## LocalSurfaceId
LocalSurfaceId consists of 3 components:
* [parent_sequence_number_](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/surfaces/local_surface_id.h;l=164;drc=0c3a07090f8fc813207632bbf6c3196b9d7e5cf1)
* [child_sequence_number_](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/surfaces/local_surface_id.h;l=165;drc=0c3a07090f8fc813207632bbf6c3196b9d7e5cf1)
* [embed_token_](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/surfaces/local_surface_id.h;l=166;drc=0c3a07090f8fc813207632bbf6c3196b9d7e5cf1)

### Sequence number
`parent_sequence_number_` and `child_sequence_number_` are both monotonically increasing number.  The former is allocated by the embedder of the client whilc the latter is allocated by the client.  
We have a allocator for each: [ParentLocalSurfaceIdAllocator](https://source.chromium.org/chromium/chromium/src/+/main:components/viz/common/surfaces/parent_local_surface_id_allocator.h) and [ChildLocalSurfaceIdAllocator](https://source.chromium.org/chromium/chromium/src/+/main:components/viz/common/surfaces/child_local_surface_id_allocator.h).  
The parent allocate the first LocalSurfaceId for the child and synchronize it when the parent-child reltionship is established. Then, the child is free to update its `child_sequence_number_`. The parent can update `parent_sequqnece_number_.`  
When the parent allocate a new id, [LayerTreeHost::SetLocalSurfaceIdFromParent](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host.cc;l=1566;drc=e735035ba1fa1a31bbb395e66bcecd4dffc002d6) is called on the child, set the passed id from parent as [CommitState::local_surface_id_from_parent](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host.cc;l=1566;drc=e735035ba1fa1a31bbb395e66bcecd4dffc002d6) of peding_commit_state and starts [DeferMainFrameUpdate](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:cc/trees/layer_tree_host.cc;l=469;drc=0c3a07090f8fc813207632bbf6c3196b9d7e5cf1). Note that this kicks BeginFrame sequence.

By having unique ids for both the embedder and the client, we can synchronize the frame between them.  
If LocalSurfaceId sent from the other component differs from what you have right now, it implies the surface is not synced yet between the embedder and the client, so needs to wait until the id becomes the same.  
While waiting, it skips BeginFrame loop and instead uses the old one not to show broken image. Not CompositorFrame is submitted. This means it won't increment the surface id at this point.

When a child-allocated LocalSurfaceID arrives in the parent, the parent updates its child_sequence_number id by [UpdateFromChild](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/viz/common/surfaces/parent_local_surface_id_allocator.cc;l=19;drc=0c3a07090f8fc813207632bbf6c3196b9d7e5cf1).

### Embed token
`embed_token_` is a UnguessbleToken(see the section below) generated by the embedder.  
This is for preventing LocalSurfaceIds to be exploited. Without this token, SurfaceId is predicable and other clients may be able to exploit this to embed surfaces they are not allowed to.  
This is why we need to generate this token on the embedder, not the client.

## UnguessableToken
[base::UnguessableToken](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/unguessable_token.h;l=49;drc=0c3a07090f8fc813207632bbf6c3196b9d7e5cf1) is a class that genertes and stores randomly chosen number as a 128 bits number.  
It is generated from a cryptographically strong random source.  
Unlike [base::Token](https://source.chromium.org/chromium/chromium/src/+/main:base/token.h), base::UnguessableToken is always generated at runtime.  
This is guaranteed by base::UngussableToken not having a method that can specify the `token_` value from outside of the class while base::Token does have a such constructor.
```cpp=
constexpr Token(uint64_t high, uint64_t low) : words_{high, low} {}
```

[UnguessbleToken::Create](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/unguessable_token.cc;l=23;drc=0c3a07090f8fc813207632bbf6c3196b9d7e5cf1) creates a token from [Token::CreateRandom], so the algorithm used to generate a random number is the same as base::Token, which is [RandomBytes](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:base/rand_util.h;l=75;drc=0c3a07090f8fc813207632bbf6c3196b9d7e5cf1) in rand_util.

