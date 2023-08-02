# Context Menu View

Context menu is a popup window you see when you, for example, right-click anywhere or click the 3-dot menu located top-right of Chrome.  
Inside a context menu, there are many items.  
Let's see how they are constructed.

## RenderViewContextMenu
[RenderViewContextMenuBase](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/renderer_context_menu/render_view_context_menu_base.h) is a class that handles context menu and its implementation for Chrome is in [RenderViewContextMenu](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/renderer_context_menu/render_view_context_menu.cc).  

[RenderViewContextMenu::InitMenu](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/renderer_context_menu/render_view_context_menu.cc;l=1009;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) constructs menu.  
You can see there are many menu groups here.  
[ContextMenuContentType::ItemGroup](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/renderer_context_menu/context_menu_content_type.h;l=25-51;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) represents a group of menu items.  
For example, [ITEM_GROUP_COPY](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/renderer_context_menu/context_menu_content_type.h;l=39;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) shows a copy-related menus like this:  
![](https://hackmd.io/_uploads/r1hiQsXs3.png)
Another example is [ITEM_GROUP_LINK](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/renderer_context_menu/context_menu_content_type.h;l=29;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) shows a link-related menus like this:   
![](https://hackmd.io/_uploads/ry-7fiXjh.png)
There are some platform-specific menus such as [ITEM_GROUP_SMART_SELECTION](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/renderer_context_menu/context_menu_content_type.h;l=30;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) which shows ARC apps that matches to the user's intentsion. For example, when user selects a address text, Google Map apps would be a good candidate to search it and open the result. This is only availble on ChromeOS.  

For each menu item group, check if it's applicable for this use case and append menu if so by RenderViewContextMenu::AppendSomething such as [AppendSmartSelectionActionItems](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/renderer_context_menu/render_view_context_menu.cc;l=1733;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227).  
On each addition, it adds [NORMAL_SEPERATOR](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/base/models/menu_separator_types.h;l=13;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) at the top and add items like this:  
```cpp=
if (menu_model_.GetItemCount())
  menu_model_.AddSeparator(ui::NORMAL_SEPARATOR);
```
This seperator is a line that you see above each group.

## How menu group is added
Some AppendHoge will call [RenderViewContextMenuObserver::InitMenu](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/base/models/menu_separator_types.h;l=13;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) if the impl is complicated and SmartSelection is one of them.  

Let's follow smart selection logic as an example.  
[StartSmartSelectionActionMenu](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/chromeos/arc/start_smart_selection_action_menu.cc;l=75;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) is its implementation.  
Here, it [RequestTextSelectionActions](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/chromeos/arc/start_smart_selection_action_menu.cc;l=105;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) to ARC to calculate the user's intent and which apps are good candidate. ARC lives in different processes, so this will be an asynchronous operation.  
Until the result comes back asynchronously from ARC, it [inserts a dummy placeholder items](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/chromeos/arc/start_smart_selection_action_menu.cc;l=118-121;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227):  
```cpp=
for (size_t i = 0; i < kMaxMainMenuCommands; ++i) {
  proxy_->AddMenuItem(IDC_CONTENT_CONTEXT_START_SMART_SELECTION_ACTION1 + i,
                      /*title=*/std::u16string());
}
```
[AddMenuItem](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/renderer_context_menu/render_view_context_menu_base.cc;l=210;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) literally adds an item at the bottom. The item to add is specified as a command id listed in [chrome_command_ids.h](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/app/chrome_command_ids.h;l=478;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) and a string as a title.  
Since this is just a placeholder now, it inserts an empty string.  

When the result becomes ready, it triggers [HandleTextSelectionActions](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:chrome/browser/chromeos/arc/start_smart_selection_action_menu.cc;l=173;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) with a vector of intent actions in `actions`.  
For each action in `actions`, it [UpdateMenuItem](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/renderer_context_menu/render_view_context_menu_proxy.h;l=97;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) and [UpdateMenuIcon](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/renderer_context_menu/render_view_context_menu_proxy.h;l=103;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227).  
If the number of actions is smaller than `kMaxMainMenuCommands`, it implies that the placeholder items are too much, so we [RemoveMenuItem](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/renderer_context_menu/render_view_context_menu_proxy.h;l=106;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227).  
And then we need to run [RemoveAdjacentSeparators](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:components/renderer_context_menu/render_view_context_menu_base.cc;l=291;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) at the end so that separator lines won't be duplicated.  

## MenuItemView
The above is how the set of menu is handled.  
Let's see how each menu is stored.

Each item is a [view](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/view.h). View is a rectangle within the View hierarchy.  

We have a special view based on ui::view for menu as well.  
[MenuItemView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/controls/menu/menu_item_view.h;l=78;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) represents a single menu item with a label and optional icon.  
It may contain [`submenu_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/controls/menu/menu_item_view.h;l=626;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227) as a child menu.  
![](https://hackmd.io/_uploads/BJOUujmj3.png)

`submenu_` also has a special view class named [SubmenuView](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/controls/menu/submenu_view.h;l=50;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227).  
Submenu may have many items, so they are contained as [`children_`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:ui/views/view.h;l=2197;drc=4e8e81f6eeb6969973f3ec97132d80339b92d227).
