# TabletState in Chromium

Tablet mode is a display mode when you flip Chromebook's monitor.  
It's the feature you can use your laptop as if it's a tablet.  
Let's see who handles tablet mode state.

## On Ash
The source of truth is in [DisplayManager::tablet_state_](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/display/manager/display_manager.h;l=721;drc=8e61c258c81aa3889231c070f8aa383e73c7a135).  
This is notified from [TabletModeController](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/wm/tablet_mode/tablet_mode_controller.h).  
[TabletModeController::SetIsInTabletPhysicalState](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/wm/tablet_mode/tablet_mode_controller.cc;l=1347;drc=8e61c258c81aa3889231c070f8aa383e73c7a135) specifies the new state calculated by [CalculateIsInTabletPhysicalState](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/wm/tablet_mode/tablet_mode_controller.cc;l=1283;drc=8e61c258c81aa3889231c070f8aa383e73c7a135).  
If [ForcePhysicalTabletState](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/wm/tablet_mode/tablet_mode_controller.h;l=217;drc=8e61c258c81aa3889231c070f8aa383e73c7a135) has a value to force, it immediately returns the forced state. This forces state is set only inside test or dev such as [kOnForTest, kOnForDev...]  
If not, check `tablet_mode_switch_is_on_` value set by [TabletModeEventReceived](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/wm/tablet_mode/tablet_mode_controller.cc;l=707;drc=8e61c258c81aa3889231c070f8aa383e73c7a135) with [PowerManagerClient::TabletMode](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/dbus/power/power_manager_client.h;l=57;drc=8e61c258c81aa3889231c070f8aa383e73c7a135).  
PowerManagerClient receives the message from dbus. It requests the tablet state to dbus by [GetSwitchStates](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/dbus/power/power_manager_client.cc;l=545;drc=8e61c258c81aa3889231c070f8aa383e73c7a135) with [OnGetSwitchStates](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/dbus/power/power_manager_client.cc;l=987;drc=8e61c258c81aa3889231c070f8aa383e73c7a135) as a callback.

## On Lacros
As mentioned above, The source of truth is in Ash side.  
Lacros receives the tablet state changes from Ash and caches the given value.  

[TabletModeController](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/wm/tablet_mode/tablet_mode_controller.h) notifies [WaylandAuraShell](https://source.chromium.org/chromium/chromium/src/+/main:components/exo/wayland/zaura_shell.cc;l=1200;drc=bf8a358fdc14c4adaa0591f107abc3ac2554389a) through [ash::TabletModeObserver](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ash/public/cpp/tablet_mode_observer.h).  
This will send wayland message [`zaura_shell_send_layout_mode`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:out/chromeos-Debug/gen/components/exo/wayland/protocol/aura-shell-server-protocol.h;l=381;drc=8e61c258c81aa3889231c070f8aa383e73c7a135). Layout mode is LAYOUT_MODE_TABLET or LAYOUT_MODE_WINDOWED.

The wayland event triggers [WaylandZauraShell::OnLayoutMode](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_zaura_shell.cc;l=103;drc=8e61c258c81aa3889231c070f8aa383e73c7a135) and passes the updated layout mode to [WaylandScreen](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_screen.cc;l=537;drc=8e61c258c81aa3889231c070f8aa383e73c7a135).  
The value is cached to [`tablet_state_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/ozone/platform/wayland/host/wayland_screen.h;l=138;drc=8e61c258c81aa3889231c070f8aa383e73c7a135) in WaylandScreen.  
This also triggers [OnDisplayTabletStateChanged](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/display/display_observer.cc;l=30;drc=8e61c258c81aa3889231c070f8aa383e73c7a135) as same as Ash side.

## Note
[chromeos::TabletState](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/base/tablet_state.h) is a class which used to store the state, but it was duplicated so now it's only used as a global singleton which can get the tablet state.  
It inner implementation of [TabletStte::state()](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/base/tablet_state.cc;l=41;drc=3b68fe280664107af07d8eeb77143b87bfa8c6dc) is only a bypass to `display::Screen::GetScreen()->GetTabletState()`.  
We can remove this class and directly checks display::Screen's `tablet_state_` value, but it's kept for now due to a historical reason.
