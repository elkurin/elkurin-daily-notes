# Views Hierarchy and Ownership

[View](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/view.h) is a rectangle withint the view hierarchy.  
All views are based on this class such as [HeaderView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/frame/header_view.h;l=45;drc=b73134cfcce34a13f25202d077c0aa9dc03b662e) and [TooltipViewAura](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/corewm/tooltip_view_aura.h;l=21;drc=63389e5d7d63cdf0e99ccff4ac7b87c2f5f264f2).  

View hierarchy is constructed of whole Browser by adding  
In this note we'll take a look in to how they are managed and owned.

# View Hierarchy
How view is stored?

Each view has [`children_`](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/view.h;l=2197;drc=508dce9dea93d32b154ed6d6d67dd29d336c4e1e) whose type is `std::vector<View>*`. Each view instance has its children views as a vector.  
This makes a hierarchy tree of [View](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/view.h).

## View Ownership
On adding view, there must be parent view.  
We call [AddChildView](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/view.h;l=442;drc=508dce9dea93d32b154ed6d6d67dd29d336c4e1e) against a view that we want to attach to. The recommended API for us to use is this:

```cpp=
template <typename T>
T* AddChildView(std::unique_ptr<T> view) {
  return AddChildView<T>(view.release());
}
```

In this API, we pass unique_ptr of the view to the parent view and it returns its raw ptr. This implies that the one who creates view is should not be the owner of each view, but instead pass the ownership to its parent. You can still use the returned raw_ptr to refer to the view.  

You can get the ownership of the view when you remove it from view hierarchy by [RemoveChildViewT](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/view.h;l=475;drc=508dce9dea93d32b154ed6d6d67dd29d336c4e1e).

```cpp=
template <typename T>
std::unique_ptr<T> RemoveChildViewT(T* view) {
  DCHECK(base::Contains(children_, view));
  RemoveChildView(view);
  return base::WrapUnique(view);
}
```

[RemoveChildViewT](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/view.h;l=475;drc=508dce9dea93d32b154ed6d6d67dd29d336c4e1e) will gives you back unique_ptr of the view.

## Implementation
As you can see from the codes, the ownership is not handled as unique_ptr.
```cpp=
template <typename T>
T* AddChildView(std::unique_ptr<T> view) {
  return AddChildView<T>(view.release());
}

template <typename T>
std::unique_ptr<T> RemoveChildViewT(T* view) {
  DCHECK(base::Contains(children_, view));
  RemoveChildView(view);
  return base::WrapUnique(view);
}
```
In [AddChildView](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/view.h;l=442;drc=508dce9dea93d32b154ed6d6d67dd29d336c4e1e), it releases unique_ptr and throw away its ownership.  
And it [base::WrapUnique](https://source.chromium.org/chromium/chromium/src/+/main:base/memory/ptr_util.h;l=16;drc=e4622aaeccea84652488d1822c28c78b7115684f) the raw_ptr to give back the ownership of the view by [RemoveChildViewT](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/view.h;l=475;drc=508dce9dea93d32b154ed6d6d67dd29d336c4e1e).  

It means unique_ptr lifetime is not managed appropriately, but instead raw_ptr inside view hierarchy is considered to have an ownerwhip inside views.

## Note
We also have [RemoveChildView](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/view.h;l=468;drc=508dce9dea93d32b154ed6d6d67dd29d336c4e1e) who takes raw_pt view as a parameter.  
This is also used as a inner implementation of [RemoveChildViewT](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/view.h;l=475;drc=508dce9dea93d32b154ed6d6d67dd29d336c4e1e). We still use this but probably we should use [RemoveChildViewT](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/view.h;l=475;drc=508dce9dea93d32b154ed6d6d67dd29d336c4e1e) as long as we can so that it's clear who has the ownership at when.
