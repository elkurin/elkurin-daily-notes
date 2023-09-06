# Hit Testing in Exo

Hit testing is a concept in Graphics to determine which unit is a target for the input event based on location.
Such as touch, click or swipe has a location and we specifies the target by locating the mouse cursor or finger upon the window or object you would like to trigger the event against.

[WindowTargeter](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.h;l=29;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) is a class to identify the target window for an event.

## Even location conversion
[WindowTargeter::EventLocationInsideBounds](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=273;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) determins whether the event is inside this input region.  
If this returns true, [SubtreeShouldBeExploedForEvent](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=31;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) can be true so that [FindTargetForLocatedEventRecursively](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=320;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) continues to search deeper in the subtree.
It calls [ConvertEventLocaltionToWindowCoordinates](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=303;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) to convert the given `event.location()` into the parent window coordinates.  
```cpp=
Window::ConvertPointToTarget(window->parent(), window, &point);
```

[ConvertPointToTarget](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window.cc;l=618;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) is a static method in aura::Window.  
It goes to [Layer::ConvertPointToLayer](https://source.chromium.org/chromium/chromium/src/+/main:ui/compositor/layer.cc;l=833;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) and transform by the scale if needed.  
The value must be int as a return value, so aura::Window get a floored scaled bounds.  

Inside [FindTargetForLocatedEventRecursively](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=320;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) which is a implementation of FindTargettForEvent, [ConvertEventToTarget](https://source.chromium.org/chromium/chromium/src/+/main:ui/events/event_target.cc;l=19;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) updates the `location_` of the event to fit in `target` coordinates.  
This modifies the `event`.



## Exo custom window targeter
Exo uses a custom window targeter extending [WindowTargeter](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.h;l=29;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c).  
Surface, SurfaceTreeHost, ShellSurfaceBase has CustomWindowTargeter for each.

All of them runs `surface->HitTest` inside [EventLocationInsideBounds](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/surface_tree_host.cc;l=68;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) to check whether it is included in the region.

[SurfaceTreeHost's CustomWindowTargeter](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/surface_tree_host.cc;l=57;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) overrides EventLocationInsideBounds and FindTargetForEvent.  
If the passed `window` is not host window, we immediately fall back to WindowTargeter's methods.  
This class is used only when the passed `window` is the host window itself.  
Host window should not be the event target, so [FindTargetForEvent](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/surface_tree_host.cc;l=84;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) returns null when the target is host window.  

[Surface's CustomWindowTargeter](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/surface.cc;l=267;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) implements EventLocationInsideBounds.  
This converts `window` to `surface` and again uses HitTest.

[ShellSurfaceBase's CustomWindowTargeter](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/shell_surface_base.cc;l=206;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) checks the current window resizing state.  
If it's under resizing step, it handles the event specially.

## HitTest
[HitTest](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/surface.cc;l=1236;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) checks whether the given `point` is contained in `hit_test_region_`.  
[`hit_test_region_`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/surface.h;l=682;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) is [cc::Region](https://source.chromium.org/chromium/chromium/src/+/main:cc/base/region.h;l=31;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c) and calculated inside [CommitSurfaceHierarchy](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/surface.cc;l=969;drc=d83e99de89c0ccc6fee4ced1e450739b142d4b2c).  
First, check `state_.basic_state.input_region` value and use the intersection between `content_size_`.  
Then calculates the union of the region for children subsurfaces similarly to `surface_hierarchy_content_bounds_`.

