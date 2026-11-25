---
title: "Plugin Class"
source: "https://plugins.exteragram.app/docs/plugin-class"
---

# Plugin Class

Understand plugin metadata, lifecycle hooks, menu items, and request or update interception.

## Metadata

Plugin metadata must be defined as plain top-level values. Do not build them dynamically, because the loader parses them from the source file with `AST`.

```python
__id__ = "better_previews"
__name__ = "Better Previews"
__description__ = "Modifies specific URLs for better previews"
__author__ = "@immat0x1"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"
__requirements__ = ["mpmath", "tinydb"]
__category__ = ["chats", "tools"]
```

Supported metadata keys:

  * `__id__`
  * `__name__`
  * `__description__`
  * `__author__`
  * `__version__`
  * `__icon__`
  * `__app_version__`
  * `__sdk_version__`
  * `__requirements__`
  * `__category__`

Legacy compatibility:

  * `__min_version__` is still accepted, but it is treated as `__app_version__ = ">=..."`.

Important details:

  * `__id__` and `__name__` are the only truly required fields.
  * `__id__` must be 2-32 characters long, start with a letter, and contain only latin letters, digits, `_`, or `-`.
  * `__requirements__` is a list of packages to install through [PIP](</docs/pip>).
  * `__description__` supports basic markdown rendering in the UI.
  * `__author__` may be plain text or a Telegram-style handle such as `@yourUsername`.
  * `__version__` defaults to `1.0` if omitted.
  * `__icon__` uses the format `StickerPackShortName/index`, for example `exteraPlugins/1`.
  * `__app_version__` supports operators such as `>=`, `<=`, `==`, `>`, and `<`.
  * `__sdk_version__` lets you require a minimum plugin SDK version when your plugin depends on newer APIs.
  * `__category__` places the plugin in the plugin catalog. Use one category as a string or up to three as a list. Unknown values are ignored.

Plugin categories:

Category| Use it for  
---|---  
`appearance`| Themes, interface tweaks, icons, and layout changes  
`chats`| Messages, chat lists, folders, and conversation features  
`media`| Photos, videos, music, voice messages, and stickers  
`privacy`| Privacy, security, and hiding information  
`tools`| Productivity helpers, automation, and utilities  
`integrations`| AI, translators, and external services  
`fun`| Games, jokes, and entertainment  
`libraries`| Shared code that other plugins depend on  
  
> > [!NOTE] **Practical baseline** > The SDK in this repository targets the current1.4.4.3API surface and expects exteraGram12.5.1+for normal initialization.

## Basic Structure

Every plugin file should define one class inheriting from `BasePlugin`.

```python
from base_plugin import BasePlugin
 
 
__id__ = "weather"
__name__ = "Weather"
 
 
class WeatherPlugin(BasePlugin):
    pass
```

## Settings

To expose a settings screen, implement `create_settings(self) -> List[Any]`.

```python
from typing import Any, List
 
from base_plugin import BasePlugin
from ui.settings import Header, Switch
 
 
class MyPlugin(BasePlugin):
    def create_settings(self) -> List[Any]:
        return [
            Header(text="General"),
            Switch(key="enabled", text="Enable feature", default=True),
        ]
```

For the full settings UI system, see [Plugin Settings](</docs/plugin-settings>).

## Plugin Events

### Load and unload

```python
from base_plugin import BasePlugin
 
 
class DebugPlugin(BasePlugin):
    def on_plugin_load(self):
        # Good place to register hooks, listeners, or background tasks.
        self.logger.info("Plugin loaded!")
 
    def on_plugin_unload(self):
        # Good place for cleanup that is not handled automatically.
        self.logger.info("Plugin unloaded!")
```

  * `on_plugin_load` runs when the plugin is enabled or restored during app startup.
  * `on_plugin_unload` runs when the plugin is disabled or the app is shutting down.

### Application events

```python
from base_plugin import AppEvent, BasePlugin
 
 
class DebugPlugin(BasePlugin):
    def on_app_event(self, event_type: AppEvent):
        if event_type == AppEvent.START:
            self.logger.info("App is starting")
        elif event_type == AppEvent.STOP:
            self.logger.info("App is stopping")
        elif event_type == AppEvent.PAUSE:
            self.logger.info("App moved to background")
        elif event_type == AppEvent.RESUME:
            self.logger.info("App returned to foreground")
```

`AppEvent` values:

  * `START`
  * `STOP`
  * `PAUSE`
  * `RESUME`

## Settings Storage Helpers

`BasePlugin` already includes small helpers for persistent plugin settings:

  * `self.get_setting(key, default=None)`
  * `self.set_setting(key, value, reload_settings=False)`
  * `self.export_settings()`
  * `self.import_settings(settings, reload_settings=True)`

```python
current_value = self.get_setting("enabled", True)
self.set_setting("enabled", not current_value, reload_settings=True)
 
backup = self.export_settings()
self.import_settings(backup, reload_settings=False)
```

## Logging

`self.logger` writes to logcat under the tag `plugin.<id>`, with levels, tracebacks and object dumps:

```python
self.logger.debug("Hook installed for %s", class_name)
 
try:
    self.process(update)
except Exception:
    self.logger.exception("Failed to process update")
 
self.logger.dump(message.messageOwner)
```

See [Logging](</docs/logging>) for levels, child loggers, timers and handlers. The older `self.log(message)` still works but is legacy.

## Menu Items

You can add menu entries to several app menus through `MenuItemData`.

```python
from typing import Any, Dict
 
from base_plugin import BasePlugin, MenuItemData, MenuItemType
 
 
class MyMenuPlugin(BasePlugin):
    def on_plugin_load(self):
        self.message_item_id = self.add_menu_item(
            MenuItemData(
                menu_type=MenuItemType.MESSAGE_CONTEXT_MENU,
                text="Log Message Info",
                on_click=self.handle_message_click,
                icon="msg_info",
                subtext="Debug helper",
                priority=10,
            )
        )
 
        self.profile_item_id = self.add_menu_item(
            MenuItemData(
                menu_type=MenuItemType.PROFILE_ACTION_MENU,
                text="Log User Info",
                on_click=self.handle_profile_click,
                icon="user_search",
            )
        )
 
    def handle_message_click(self, context: Dict[str, Any]):
        self.logger.info(f"Message menu item clicked. Context keys: {list(context.keys())}")
 
        message = context.get("message")
        if message:
            self.logger.info(f"Clicked on message ID: {message.getId()}")
            self.logger.info(f"Message text: {message.messageText}")
 
    def handle_profile_click(self, context: Dict[str, Any]):
        self.logger.info(f"Profile menu item clicked. Context keys: {list(context.keys())}")
 
        user = context.get("user")
        if user:
            self.logger.info(f"Profile opened for user: {user.first_name} (ID: {user.id})")
```

### MenuItemData

`MenuItemData` fields:

  * `menu_type: MenuItemType` \- required target menu
  * `text: str` \- required visible title
  * `on_click: Callable[[Dict[str, Any]], None]` \- required click handler
  * `item_id: Optional[str]` \- optional stable id if you want to remove the item later
  * `icon: Optional[str]` \- drawable name, such as `"msg_info"`
  * `subtext: Optional[str]` \- secondary line below the title
  * `condition: Optional[str]` \- MVEL expression which controls visibility
  * `priority: int` \- larger values usually appear earlier

### MenuItemType

Current menu types exposed by the SDK:

  * `MenuItemType.MESSAGE_CONTEXT_MENU`
  * `MenuItemType.DRAWER_MENU`
  * `MenuItemType.MAIN_MENU`
  * `MenuItemType.CHAT_ACTION_MENU`
  * `MenuItemType.PROFILE_ACTION_MENU`

### The `on_click` Context

The callback receives a dictionary with context-dependent values.

Common keys include:

  * `account`
  * `context`
  * `fragment`
  * `dialog_id`
  * `user`
  * `userId`
  * `userFull`
  * `chat`
  * `chatId`
  * `chatFull`
  * `encryptedChat`
  * `message`
  * `groupedMessages`
  * `botInfo`

Because the exact set depends on the menu type and screen, it is best to inspect the available keys during development:

```python
def handle_click(self, context):
    self.logger.info(f"Available keys: {list(context.keys())}")
```

### Removing Menu Items

Menu items are removed automatically when the plugin unloads, but you can also remove one manually:

```python
if self.message_item_id:
    self.remove_menu_item(self.message_item_id)
```

## Event Hooks

Event hooks let your plugin intercept Telegram requests, responses, updates, or outgoing message parameters.

You register them with:

  * `self.add_hook(name, match_substring=False, priority=0)`
  * `self.add_on_send_message_hook(priority=0)`

```python
class GhostModePlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_hook("TL_messages_setTyping")
        self.add_hook("TL_messages_setEncryptedTyping")
        self.add_hook("TL_account_updateStatus")
        self.add_on_send_message_hook()
```

Every hook receives the `account` it fired for, and it is often **not** the account selected in the UI — background accounts stay connected and keep firing hooks. Modifying the hooked object, as the examples below do, needs no special care. But the moment you send, request, or read something in response, use `self.client(account)` or an explicit `account=`, or you will act on whichever account happens to be on screen. See [Multi-account](</docs/multi-account>).

Hook results are returned as `HookResult` with a `HookStrategy`.

```python
from typing import Any, List
 
from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Switch
 
 
TYPING_REQUESTS = ["TL_messages_setTyping", "TL_messages_setEncryptedTyping"]
 
 
class GhostModePlugin(BasePlugin):
    def on_plugin_load(self):
        for req_name in TYPING_REQUESTS:
            self.add_hook(req_name)
 
        self.add_hook("TL_account_updateStatus")
 
    def pre_request_hook(self, request_name: str, account: int, request: Any) -> HookResult:
        if request_name in TYPING_REQUESTS and self.get_setting("dont_send_typing", True):
            self.logger.info(f"Blocking request: {request_name}")
            return HookResult(strategy=HookStrategy.CANCEL)
 
        if request_name == "TL_account_updateStatus" and self.get_setting("force_offline", True):
            request.offline = True
            return HookResult(strategy=HookStrategy.MODIFY, request=request)
 
        return HookResult()
 
    def post_request_hook(self, request_name: str, account: int, response: Any, error: Any) -> HookResult:
        if request_name == "TL_messages_sendMessage" and not error:
            self.logger.info("Successfully sent a message!")
        return HookResult()
 
    def create_settings(self) -> List[Any]:
        return [
            Switch(key="dont_send_typing", text="Don't send typing status", default=True),
            Switch(key="force_offline", text="Always appear offline", default=True),
        ]
```

### Hook strategies

  * `HookStrategy.DEFAULT`: do nothing special
  * `HookStrategy.CANCEL`: stop the operation
  * `HookStrategy.MODIFY`: return a modified object
  * `HookStrategy.MODIFY_FINAL`: return a modified object and stop further plugin processing for that event

Depending on the hook type, the modified object should be assigned to:

  * `HookResult.request`
  * `HookResult.response`
  * `HookResult.update`
  * `HookResult.updates`
  * `HookResult.params`

### Update hooks

```python
def on_update_hook(self, update_name: str, account: int, update: Any) -> HookResult:
    result = HookResult()
 
    if update_name == "TL_updateNewMessage":
        self.logger.info(f"Intercepted update: {update_name}")
 
    return result
 
 
def on_updates_hook(self, container_name: str, account: int, updates: Any) -> HookResult:
    result = HookResult()
 
    if container_name == "TL_updates":
        self.logger.info(f"Received updates container: {container_name}")
 
    return result
```

### Outgoing message hook

```python
def on_send_message_hook(self, account: int, params: Any) -> HookResult:
    result = HookResult()
 
    if hasattr(params, "message") and isinstance(params.message, str):
        if params.message.startswith(".hello"):
            params.message = params.message.replace(".hello", "Hello from plugin")
            result.strategy = HookStrategy.MODIFY
            result.params = params
 
    return result
```

## Xposed Method Hooking

For low-level Java method hooking through LSPosed/Xposed-style APIs, see the dedicated [Xposed Method Hooking](</docs/xposed-hooking>) page.

[PreviousTroubleshooting](</docs/elyx/troubleshooting>)[NextMulti-account](</docs/multi-account>)
