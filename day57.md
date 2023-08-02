# Browser Manager

[BrowserManager](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/browser_manager.h;drc=fe503e7168a2752a0dcf97c580b347a2fac094fb) is a key class for Lacros. It manages the lifetime of lacros-chrome and its loading status, checks the versions and observes the component updater for future updates etc..  
There are many responsibilities for [BrowserManager](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/browser_manager.h;drc=fe503e7168a2752a0dcf97c580b347a2fac094fb) class.  
Let's list up its roles.

## Overview
[BrowserManager](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/browser_manager.h;drc=fe503e7168a2752a0dcf97c580b347a2fac094fb) is a singleton living in Ash.  
It's held as `g_instance` in browser_manager.cc and we can obtain its instance from [BrowserManager::Get](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=568;drc=6b569de26935ca8c8af6e78206f675beca37c27d).  

## Lacros startup/shutdown
[Start](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1012;drc=6b569de26935ca8c8af6e78206f675beca37c27d) triggers the lacros-chrome loading.  
It will [WaitForDeviceOwnerFetchedAndThen](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1681;drc=6b569de26935ca8c8af6e78206f675beca37c27d) runs lacros-chrome with creating a log file from [StartWithLogFile](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1052;drc=6b569de26935ca8c8af6e78206f675beca37c27d).  
Here, BrowserLoader creates a process for Lacros with [command line](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1138;drc=6b569de26935ca8c8af6e78206f675beca37c27d). It [CreateStartupData](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1138;drc=6b569de26935ca8c8af6e78206f675beca37c27d) to get start upda data.  
Before launching Lacros process, BrowserLoader sends an invitation for mojo connection from [CrosapiManager::SendInvitation](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1276;drc=6b569de26935ca8c8af6e78206f675beca37c27d) and then create lacros subprocess is launched [here](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1301;drc=6b569de26935ca8c8af6e78206f675beca37c27d).  

Lacros termination will be catched by the mojo disconnection.  
[OnMojoDisconnected](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1398;drc=6b569de26935ca8c8af6e78206f675beca37c27d) will call [HandleLacrosChromeTermination](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1404;drc=6b569de26935ca8c8af6e78206f675beca37c27d).  

Or we can [Shutdown](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=930;drc=6b569de26935ca8c8af6e78206f675beca37c27d) lacros from BrowserManager.  
First disable keep alive and then `lacros_process_.Terminate`.   [HandleLacrosChromeTermination](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1404;drc=6b569de26935ca8c8af6e78206f675beca37c27d) will be called after this.  


### States

Here' are the lanching status for lacros:
[State](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/browser_manager.h;l=416-452;drc=fe503e7168a2752a0dcf97c580b347a2fac094fb) has:
* NOT_INITIALIZED
* RELOADING
* MOUNTING
* UNAVAILABLE
* STOPPED
* CREATING_LOG_FILE
* PRE_LAUNCHED
* STARTING
* RUNNING
* TERMINATING

## Open Window
There are many APIs in BrowserManager to create a window such as:  
* [NewWindow](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/browser_manager.h;l=156;drc=fe503e7168a2752a0dcf97c580b347a2fac094fb)
* [OpenForFullRestore](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/browser_manager.h;l=161;drc=fe503e7168a2752a0dcf97c580b347a2fac094fb)
* [OpenUrl](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.h;l=206;drc=6b569de26935ca8c8af6e78206f675beca37c27d)

These APIs are for opening specified url or about://blank page in a new window.  

Also it handles tabs such as:
* [NewTab](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/browser_manager.h;l=196;drc=fe503e7168a2752a0dcf97c580b347a2fac094fb)
* [SwitchToTab](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.h;l=217;drc=6b569de26935ca8c8af6e78206f675beca37c27d)
* [HandleTabScrubbing](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.h;l=226;drc=6b569de26935ca8c8af6e78206f675beca37c27d)

These APIs create [BrowserAction](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_action.h) and [PerformOrEnqueue](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1937;drc=6b569de26935ca8c8af6e78206f675beca37c27d) it.  
If `state_` is ready, Perform the action.  
If it's starting or such, it pushes `action` to `pending_actions_` (enqueue).  
If not, it writes a log accordingly to why it can not do such action.  

## Version Control
[BrowserVersionServiceDelegate](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=485;drc=6b569de26935ca8c8af6e78206f675beca37c27d) keeps track of the most recent lacros-chrome binary version loaded by [BrowserLoader](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/browser_loader.h).  

BrowserManager stores the version value in [`browser_version_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.h;l=745;drc=6b569de26935ca8c8af6e78206f675beca37c27d) and which of rootfs or stateful was picked in [`lacros_selection_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.h;l=739;drc=6b569de26935ca8c8af6e78206f675beca37c27d).  
