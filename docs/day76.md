# BrowserManager State Transition

[BrowserManager](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/browser_manager.h;l=101;drc=50645a5536e3c3eb8c1782716db2bb97db33ae93) is a critical class that controls lacros-chrome.  
The logic to launch lacros is complicated, and the current step that BrowserManager is taking is controlled as State.  
Let's look at how the state changes on BrowserManager.

## BrowserManager flowchart
[BrowserManager::State](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.h;l=392;drc=a9163a67bbe920dfcfa6286bf0d172a1af377dba) represents a current state of BrowserManager.  
There are 10 states: NOT_INITIALIZED, RELOADING, MOUNTING, UNAVAILABLE, STOPPED, CREATING_LOG_FILE, STARTING, RUNNING, TERMINATING.  

Here's the state transition:  
![](https://hackmd.io/_uploads/Hy6-8GW2n.png)

The current state is stored as [`state_`](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ash/crosapi/browser_manager.h;l=697;drc=50645a5536e3c3eb8c1782716db2bb97db33ae93).  
[SetState](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=969;drc=a9163a67bbe920dfcfa6286bf0d172a1af377dba) will be called to update the state when neccessary.

## PerformOrEnqueue
On triggering browser related actions, it's requested to BrowserManager.  
It is called [BrowserAction](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_action.h;l=24;drc=a9163a67bbe920dfcfa6286bf0d172a1af377dba), a base class of the browser actions such as opening new window, restoring tabs, tab scrubbing...  

BrowserManager decides whether the requested action should run immediately or enqueued to run later, or cancel.  
[PerformOrEnqueue](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1949;drc=a9163a67bbe920dfcfa6286bf0d172a1af377dba) checks current BrowserManager's state.  

If it's RUNNING, we can [Perform](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_action.h;l=80;drc=a9163a67bbe920dfcfa6286bf0d172a1af377dba) an action.  

If its UNAVAILABLE or `shutdown_requested_` Cancel an action.  

For other cases, we enqueue the passed action to `pending_actions_` and run them in queued order when [the state becomes RUNNING](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1370;drc=a9163a67bbe920dfcfa6286bf0d172a1af377dba).  
Some BrowserAction should not be queued. For example, [HandleTabScrubbingAction](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ash/crosapi/browser_manager.cc;l=1370;drc=a9163a67bbe920dfcfa6286bf0d172a1af377dba) does not mean much when the browser window is not yet ready, so it sets `is_queueable_` false to skip queueing.

TODO(elkurin): Can we migrate `shutdown_requested_` as one of the State?  



