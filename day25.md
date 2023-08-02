# Wayland Library - How message is sent from Server + alpha
In [the previous note](https://hackmd.io/@elkurin/ryt69lhwh), I went through how message is called by Client.  
Today, let's go through the path from Server to Client.

## Server Implementation
From server, the message is treated as event.  
For example, let's look at the message sent from server when it closes tooltip.
```cpp=
static inline void
zaura_surface_send_tooltip_hidden(struct wl_resource *resource_)
{
  wl_resource_post_event(resource_, ZAURA_SURFACE_TOOLTIP_HIDDEN);
}
```
This is in a generated file from [aura-shell.xml](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/aura-shell.xml).  

On calling [zaura_surface_send_tooltip_hidden](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:out/chromeos-Debug/gen/components/exo/wayland/protocol/aura-shell-server-protocol.h;l=1175;drc=8d3b19d9432fa06963a1efbd8b85d5d310c3a05a), it triggers [wl_resource_post_event](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-server.c;l=249;drc=8d3b19d9432fa06963a1efbd8b85d5d310c3a05a).  
Similarly to Client call, the message is identified by opcode such as [`ZAURA_SURFACE_TOOLTIP_HIDDEN`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:out/chromeos-Debug/gen/components/exo/wayland/protocol/aura-shell-server-protocol.h;l=916;drc=8d3b19d9432fa06963a1efbd8b85d5d310c3a05a).

These parameters are passed to [`handle_array`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-server.c;l=210;drc=090cbdd744cf1b4f1e74caac6c0f0ea78cdc594e).
Similarly to client side, it constructs `wl_closure` from `wl_closure_marshal`. Instead of passing [`interface->method`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-client.c;l=836;drc=090cbdd744cf1b4f1e74caac6c0f0ea78cdc594e), server implementation gets [`interface->events`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-server.c;l=226;drc=090cbdd744cf1b4f1e74caac6c0f0ea78cdc594e) as a message to send.  
And then call [`wl_closure_send`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=1357;drc=090cbdd744cf1b4f1e74caac6c0f0ea78cdc594e) to call closure.  
From here, this is same to Client side impl.

## Client Implementation
[`TooltipHidden`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_zaura_surface.cc;l=290;drc=090cbdd744cf1b4f1e74caac6c0f0ea78cdc594e) is the implementation of the event "tooltip_hidden".
How is it registered as `interface->events`?

[WaylandZAuraSurface](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_zaura_surface.cc;l=13;drc=090cbdd744cf1b4f1e74caac6c0f0ea78cdc594e) creates `zaura_surface_listener` list including [`TooltipHidden`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_zaura_surface.cc;l=290;drc=090cbdd744cf1b4f1e74caac6c0f0ea78cdc594e).
[`zaura_surface_add_listener`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:out/Debug/gen/components/exo/wayland/protocol/aura-shell-client-protocol.h;l=718;drc=090cbdd744cf1b4f1e74caac6c0f0ea78cdc594e) -> [`wl_proxy_add_listener`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-client.c;l=578;drc=090cbdd744cf1b4f1e74caac6c0f0ea78cdc594e) adds `zaura_surface_listener` to [`object.implementation`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-client.c;l=589;drc=090cbdd744cf1b4f1e74caac6c0f0ea78cdc594e).
This `implementation` is refered from [`wl_closure_invoke`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=1145;drc=f80633b34538615fcb73515ad8c4bc56a748abfe) and invoked by server.  

## How xml file generates protocol files
We mentioned "generated file" several times. Those protocol files are generated from xml files like [aura-shell.xml](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/aura-shell.xml).  

The format is like this:
```
<protocol name="aura_shell">
  ...
  <interface name="zaura_surface" version="51">
    ...
    <event name="tooltip_hidden" since="47">
      <description summary="tooltip is hidden by server side">
        Informs the client that the tooltip is hidden.
      </description>
    </event>
    ...
  </interface>
  ...
</protocol>
```
How are they generated?

aura-shell.xml is built by [wayland_protocol](https://source.chromium.org/chromium/chromium/src/+/main:third_party/wayland/wayland_protocol.gni) source set [here](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/protocol/BUILD.gn;l=7;drc=8ba1bad80dc22235693a0dd41fe55c0fd2dbdabd).  
[wayland_protocol](https://source.chromium.org/chromium/chromium/src/+/main:third_party/wayland/wayland_protocol.gni) template decides protocol output file locations to generate.  
```python=
protocol_outputs += [
  "${dir}/${name}-protocol.c",
  "${dir}/${name}-server-protocol.h",
  "${dir}/${name}-client-protocol.h",
]
```

And runs scanner in [wayland_scanner_wrapper.py](https://source.chromium.org/chromium/chromium/src/+/main:third_party/wayland/wayland_scanner_wrapper.py) to generate code.  
It runs [scanner.c](https://source.chromium.org/chromium/chromium/src/+/main:third_party/wayland/src/src/scanner.c) in subprocess.  

## Note
### Maximum args
According to [`WL_CLOSURE_MAX_ARGS`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-private.h;l=49;drc=090cbdd744cf1b4f1e74caac6c0f0ea78cdc594e), the maximum args for the message is 20.

### va_list / va_start
During reading server side code, there was `va_*`.  
[`va_list`](https://cplusplus.com/reference/cstdarg/va_list/) is a type to hold information about variable arguments of a function.  
It is included in [cstdarg](https://en.cppreference.com/w/cpp/header/cstdarg) standard library.
