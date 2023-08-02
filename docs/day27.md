# Client and Non-Client views

Widget, NonClientView, NonClientFrameView, HeaderView somethingView and so on.  
There are too many views and frames and thier responsibility is complicated.
Let's list up them in this note.

## What is NonClient?

"NonClient" means it's not the image sent from client such as blink or applications.  
For example, widnow decorations, Widget border, the shadow are included in NonClientFrameView. NonClient view mostly does not depend on the image or frames to show.  
On the other hand, "Client" view is an actual content.


## Overview
There is a great image in [Views Overview](https://chromium.googlesource.com/chromium/src/+/master/docs/ui/views/overview.md) doc.  

![](https://hackmd.io/_uploads/BJ6iHSg_2.png)

Citing from [Views Overview](https://chromium.googlesource.com/chromium/src/+/master/docs/ui/views/overview.md)

This image shows the boundary inclusion.  
Widget includes all the visible frames.  
Note that there could be a case where some frame crosses the bounrdary of widget such as BubbleView or overlay frame, but we forget about those cases in this image.

## Widget hierarchy in Views
Widget owns internal::RootView as [`root_view_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/widget.h;l=1295;drc=51e1b713f6da38219910bf8fb93a81262340bf97)
For BrowserFrame, [BrowserView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/ui/views/frame/browser_root_view.h;l=27;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16) is a RootView implementation.  

On Widget::Init, it initializes [`root_view_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/widget.h;l=1295;drc=51e1b713f6da38219910bf8fb93a81262340bf97) and [`non_client_view_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/widget.h;l=1301;drc=51e1b713f6da38219910bf8fb93a81262340bf97) [SecContentsView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/widget/widget.cc;l=454;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16) to it.  

[NonClientView](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/window/non_client_view.h;l=151;drc=75d7c513ef4be3f9d6ee591fc038359c90949aae) is the logical root of all views contained within a Window.  

There are 2 views inside [NonClientView](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/window/non_client_view.h;l=151;drc=75d7c513ef4be3f9d6ee591fc038359c90949aae) as direct children.  
- [NonClientFrameView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/window/non_client_view.h;l=29;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16)
- [ClientView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/window/client_view.h;l=24;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16)

[NonClientFrameView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/window/non_client_view.h;l=29;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16) is a View that renders and responds to events within the frame portions of the non client area of a window.  

[ClientView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/window/client_view.h;l=24;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16) occupies the "clint area" of a widget.  
Inside ClientView, [`contents_view_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/window/client_view.h;l=76;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16) represents a real content. This size does not include boundary who has a width of 1px.  
Therefore, the size of [`contents_view_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/window/client_view.h;l=76;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16) is 2px smaller than widget bounds for both width and height.


## //chromeos/ui frames
The above was a story inside Views.  
Here, we focus on ChromeOS implementation.

[NonClientFrameViewBase](https://source.chromium.org/chromium/chromium/src/+/main:chromeos/ui/frame/non_client_frame_view_base.h) is an inheritting class of [NonClientFrameView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/window/non_client_view.h;l=29;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16).  

The below are example of views used in //chromeos.

- [HeaderView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/frame/header_view.h;l=45;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16) paints the frame header such as title, caption uttons.
This is constructed and set to [NonClientFrameViewBase](https://source.chromium.org/chromium/chromium/src/+/main:chromeos/ui/frame/non_client_frame_view_base.h). This view represents the header such as title, caption button at the top-right.
- [HeaderContentView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/frame/header_view.cc;l=38;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16)
Owned by [HeaderView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/frame/header_view.h;l=45;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16). Used to draw the content in header, background and title string.
- [OverlayView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/frame/non_client_frame_view_base.h;l=81;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16)
This contains entire widget including [HeaderView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/frame/header_view.h;l=45;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16) as a child. As implemented in [OverlayView::Layout](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/frame/non_client_frame_view_base.cc;l=31;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16), HeaderViews bounds is set to fit inside overlay view.
- [DefaultFrameHeader](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chromeos/ui/frame/default_frame_header.h;l=22;drc=3e67c0491457f1a05fb7816346fc6d9676bb7a16)
Provides header for Chrome apps.  

## Other views
There is a [BubbleFrameView](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/bubble/bubble_frame_view.h). Bubble view is similar to popup, but different as popup has its own window, compositor and frame tree sink while bubble view is tied with toplevel window (widget). Bubble view can be seen for example on hovering over the tab at the top of the window.

[Border](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/border.h) display a boarder around a view.  

[Background](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/background.h;drc=682129de1dbc24ebc1aa3c8b3d160cb13c0acaa6) paints a background but I'm not sure where "background" is. Will read later.
