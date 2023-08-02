# Wayland version

Server and Client may have different versions. In such case, there is a case where newly introduced API is not yet supported. We need to check the availability before calling APIs.  

How versions are controlled on wayland protocol?  

## Versions
Versions in protocol is handled as an integer which increments one by one.
It's written like this:  
```
<interface name="zaura_shell" version="57">
...
  <request name="get_aura_surface">...</request>
  <request name="get_aura_output" since="2">...</request>
```
Each interface has an integer version. This integer is incremented manually by developers when they add new protocol.  
On adding a request or event, add a parameter "since" and specifiy the version when the protocol is supported.  

This since version is registered as a macro generated on xx-protocol.h  
For example `ZAURA_TOPLEVEL_SURFACE_SUBMISSION_IN_PIXEL_COORDINATES_SINCE_VERSION` in aura-shell-client-protocol.h and aura-shell-server-protocol.h. Macros has the same number on both server and client protocol.  

## Setting Version
Both Client and Server sets a version that they support.  

### Client
Wayland connection is established by [WaylandConnection::Initialize](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_connection.cc;l=133;drc=a742c65cc620d2b7b262b5514540ebdb510a7121). Inside it, it registers [WaylandZAuraShell::Instantiate](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_zaura_shell.cc;l=32;drc=a742c65cc620d2b7b262b5514540ebdb510a7121) and other instantiate functions.  
On binding zaura_shell by [wl::Bind](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/common/wayland_object.h;l=67;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687), we set the version number.  
```cpp=
auto zaura_shell = wl::Bind<struct zaura_shell>(
  registry, name, std::min(version, kMaxVersion));
```

[`kMaxVersion`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_zaura_shell.cc;l=25;drc=c3f32fac9e8a61bf1e1c54c1022a96af097cf687) number is incremented by a developer when the implemented API become supported.  

### Server
Similarly to Client side, [`bind_aura_shell`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/exo/wayland/zaura_shell.cc;l=1713;drc=a742c65cc620d2b7b262b5514540ebdb510a7121) create `zaura_shell_interface` resource with specifying a version.  

[`kZAuraShellVersion`](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/zaura_shell.h;l=36;drc=574ea61379eff905b63db07b407f060f275aac10) is a maximum number supported on server side for zaura shell.  

## Getting Version
### Client getting Server's version
It works like this:
```cpp=
if (augmented_surface_get_version(get_augmented_surface()) >=
    AUGMENTED_SURFACE_SET_ROUNDED_CORNERS_CLIP_BOUNDS_SINCE_VERSION) {
```
You can get version by passing the interface instance to xx_get_version protocol and compare it with the macro since version.  

### Server getting Client's version
It works like this:
```cpp=
void AuraSurface::OnFrameLockingChanged(Surface* surface, bool lock) {
  if (wl_resource_get_version(resource_) <
      ZAURA_SURFACE_LOCK_FRAME_NORMAL_SINCE_VERSION)
    return;
```
You can obtain the version via `wl_resource_get_version` and compare it with the macro since version as same as client side.
