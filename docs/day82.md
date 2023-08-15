# WindowTargeter
Let's investigate the event targeting logic.

## What is WindowTargeter
[WindowTargeter](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.h) is a class to find a target for the event.  
This is used when the event is dispatched to some area and we need to find which window is targeted by the event.

[Event](https://source.chromium.org/chromium/chromium/src/+/main:ui/events/event.h) is a class that represents event.  
There are many types of event such as [LocatedEvent](https://source.chromium.org/chromium/chromium/src/+/main:ui/events/event.h;l=345;drc=26d782c8efa9b35836e4e96e45a60326a5a4eed4), [KeyEvent](https://source.chromium.org/chromium/chromium/src/+/main:ui/events/event.h;l=787;drc=26d782c8efa9b35836e4e96e45a60326a5a4eed4), [MouseEvent](https://source.chromium.org/chromium/chromium/src/+/main:ui/events/event.h;l=433;drc=c8a629d7be311323904c523d33693af8f52d17e9) which extends LocatedEvent, [ScrollEvent](https://source.chromium.org/chromium/chromium/src/+/main:ui/events/event.h;l=977;drc=26d782c8efa9b35836e4e96e45a60326a5a4eed4) which extends MouseEvent and so on.  

## How to find target
[FindTargetForEvent](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=153;drc=c8a629d7be311323904c523d33693af8f52d17e9) is called to find the target.  
`event` is the dispatched event, and `root` is where the event is dispatched at.  
At this point, we are not sure which window inside the window tree hierarchy under `root` is a target, and [WindowTargeter](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.h) is responsible for finding it.  
`root` is EventTargeter class while [aura::Window](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window.h;l=106;drc=c8a629d7be311323904c523d33693af8f52d17e9) inherits [EventTarget](https://source.chromium.org/chromium/chromium/src/+/main:ui/events/event_target.h;l=26;drc=c8a629d7be311323904c523d33693af8f52d17e9), so `root` can be casted into aura::Window.

If `event` is a key event, it triggers [FindTargetForKeyEvent](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=166;drc=c8a629d7be311323904c523d33693af8f52d17e9).  
For key event, the [focused client](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=168;drc=c8a629d7be311323904c523d33693af8f52d17e9) is the one the event should be dispatched at.  
Focused client can be obtained as a value property of [kFocusClientKey](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/client/aura_constants.cc;l=57;drc=c8a629d7be311323904c523d33693af8f52d17e9) attached to the window.  
For non-key event, [FindTargetForNonKeyEvent](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=312;drc=c8a629d7be311323904c523d33693af8f52d17e9) is called.  
If the event is not [LocatedEvent](https://source.chromium.org/chromium/chromium/src/+/main:ui/events/event.h;l=227;drc=c8a629d7be311323904c523d33693af8f52d17e9), it immediately handles `root_window` as a target. Located event is mouse event, scroll event, touch event or gesture event.  
(Event types are listed in [EventType](https://source.chromium.org/chromium/chromium/src/+/main:ui/events/types/event_type.h;l=12;drc=c8a629d7be311323904c523d33693af8f52d17e9) enum)  
If not, it triggers [FindTargetForLocatedEvent](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=188;drc=c8a629d7be311323904c523d33693af8f52d17e9).  
It first checks whether `root_window` is suitable through [FindTargetInRootWindow](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=109;drc=c8a629d7be311323904c523d33693af8f52d17e9). If it is, `root_window` becomes the final result of window target.  
If it returns nullptr, then we start checking recursively with [FindTargetForLocatedEventRecursively](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window_targeter.cc;l=320;drc=c8a629d7be311323904c523d33693af8f52d17e9).  
It gets `root_window->GetChildIterator()` and traverse the children windows recursively.  
Finally, check whether the potential target window can accept event with [Window::CanAcceptEvent](https://source.chromium.org/chromium/chromium/src/+/main:ui/aura/window.cc;l=1476;drc=c8a629d7be311323904c523d33693af8f52d17e9).  
This could be false for example when it's invisible.
