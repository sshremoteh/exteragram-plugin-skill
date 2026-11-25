# exteraGram Plugin Development: Complete Reference Guide for LLMs

> **Цель документа**: Служить полной контекстной базой знаний для нейросетей при генерации плагинов для exteraGram.
> **Версия SDK**: 1.4.4.3+ (Baseline app: exteraGram 12.5.1+, Python 3.11, Chaquopy v16, Xposed-style Hooking)

---



# === DOCUMENT: Introduction (/docs) ===

---
title: "Introduction"
source: "https://plugins.exteragram.app/docs"
---

# Introduction

Plugin development in the language most Telegram modders already use.

# exteraGram Plugins

exteraGram plugins are written in Python and run inside the app through the built-in plugin engine.

The SDK in this repository is powered by [Chaquopy v16](<https://chaquo.com/chaquopy/doc/16.0/>) for Python interop and Xposed-style method hooking.

## Current baseline

  * Current SDK version in this repository: `1.4.4.3`
  * Python runtime: `3.11`
  * Recommended minimum app version for this SDK: `exteraGram 12.5.1+`

> > [!NOTE] **App version matters** > If the app build is too old, the SDK enters safe mode and plugin modules refuse to initialize. In practice, treat12.5.1+as the baseline when writing or testing new plugins against this tree.

## What you can do with the SDK

  * register high-level plugin lifecycle hooks through [`BasePlugin`](</docs/plugin-class>)
  * intercept requests, responses, updates, and outgoing messages
  * create plugin settings pages and custom settings rows
  * show Telegram-style dialogs and bulletins
  * send or edit messages and media
  * reflect into Java classes and fields
  * hook Java methods and constructors
  * generate Java proxy classes from Python with [`class proxy`](</docs/class-proxy>)

## Recommended reading order

  1. [Setup](</docs/setup>)
  2. [First Plugin](</docs/first-plugin>)
  3. [Plugin Class](</docs/plugin-class>)
  4. [Plugin Settings](</docs/plugin-settings>)
  5. [Client Utilities](</docs/client-utils>)

## Main modules at a glance

  * `base_plugin`: metadata, lifecycle, event hooks, menu items, Xposed helpers
  * `android_utils`: UI thread helpers, listener wrappers, logging, clipboard
  * `client_utils`: requests, controllers, queues, sending and editing messages
  * `file_utils`: directories plus simple file I/O helpers
  * `hook_utils`: reflection helpers for classes and private fields
  * `ui.*`: dialogs, bulletins, settings
  * `extera_utils.*`: class proxy DSL, text formatting helpers, metadata utilities

[NextSetup](</docs/setup>)


---


# === DOCUMENT: Setup (/docs/setup) ===

---
title: "Setup"
source: "https://plugins.exteragram.app/docs/setup"
---

# Setup

How to start developing plugins.

### Install a recent app build

Make sure you are using a recent build of exteraGram with plugin support enabled.

For this SDK tree, treat `exteraGram 12.5.1+` as the practical minimum.

Current test builds are usually published in the [beta channel](<https://t.me/exteraGramCI>).

### Enable the plugin engine

After logging into your account, go to `exteraGram Preferences` > `Plugins` and enable the plugin engine.

Then open the info screen for plugins and enable developer mode.

### Create a local project

Create a folder on your PC and add a Python file for the plugin.

A simple layout is enough:

```python
my-plugin/
`-- hello_world.py
```

Recommended desktop-development tips:

  * keep the filename stable; the least surprising setup is `filename == __id__ + ".py"`
  * point your IDE at this repository's `src/` directory for imports and autocomplete
  * keep metadata as plain top-level constants so the loader can parse them

Minimal starter file:

```python
from base_plugin import BasePlugin
 
__id__ = "hello_world"
__name__ = "Hello World"
__description__ = "My first exteraGram plugin"
__author__ = "Your Name"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"
 
 
class HelloWorldPlugin(BasePlugin):
    pass
```

### Connect to the device

Connect your phone to your PC and make sure [ADB](<https://developer.android.com/tools/adb>) is available on your `PATH`.

You have two common workflows:

  * copy the `.py` file into the app's plugins directory and reload it manually
  * use your own helper tooling which talks to the built-in [`DevServer`](</docs/dev-server>) for write/reload/debug flows

On-device plugin files live under a path similar to:

```python
/data/user/0/com.exteragram.messenger/files/plugins/<plugin_id>.py
```

> > [!NOTE] **Debugging ports** > The built-in development server listens on42690by default. The debugger port is separate and chosen by you or your tooling. In the example below it is5678.

VS Code remote attach example:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python Debugger: Attach exteraGram plugin",
            "type": "debugpy",
            "request": "attach",
            "connect": {
                "host": "localhost",
                "port": 5678
            },
            "pathMappings": [
                {
                    "localRoot": "/path/to/your/workspace",
                    "remoteRoot": "/data/user/0/com.exteragram.messenger/files/plugins"
                }
            ]
        }
    ]
}
```

Inside that remote directory, your plugin file should still match the plugin id, for example `hello_world.py`.

## First pages to keep open

  * [First Plugin](</docs/first-plugin>)
  * [Plugin Class](</docs/plugin-class>)
  * [Plugin Settings](</docs/plugin-settings>)
  * [Client Utilities](</docs/client-utils>)

[PreviousIntroduction](</docs>)[NextFirst Plugin](</docs/first-plugin>)


---


# === DOCUMENT: First Plugin (/docs/first-plugin) ===

---
title: "First Plugin"
source: "https://plugins.exteragram.app/docs/first-plugin"
---

# First Plugin

Build and test your first real plugin.

This page walks through a realistic first plugin for the current SDK.

We will build a small command plugin:

  * when the user sends `.hello Alice`, the plugin rewrites the outgoing message
  * the greeting template is configurable in plugin settings
  * the example stays synchronous and easy to read on purpose

## Before you start

Keep these pages nearby:

  * [Plugin Class](</docs/plugin-class>)
  * [Plugin Settings](</docs/plugin-settings>)
  * [Client Utilities](</docs/client-utils>)

## Minimal plugin shape

Every plugin file should contain:

  * metadata as plain top-level constants
  * one class inheriting from `BasePlugin`

```python
from base_plugin import BasePlugin
 
__id__ = "hello_world"
__name__ = "Hello World"
__description__ = "My first exteraGram plugin"
__author__ = "Your Name"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"
 
 
class HelloWorldPlugin(BasePlugin):
    pass
```

## Complete first plugin

```python
from typing import Any, List
 
from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Header, Input, Text
 
__id__ = "hello_world"
__name__ = "Hello World"
__description__ = "Rewrites .hello commands into a friendly message"
__author__ = "Your Name"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"
 
 
DEFAULT_TEMPLATE = "Hello, {name}!"
 
 
class HelloWorldPlugin(BasePlugin):
    def on_plugin_load(self):
        # Register the outgoing-message hook once when the plugin is loaded.
        self.add_on_send_message_hook()
        self.logger.info("Hello World plugin loaded")
 
    def create_settings(self) -> List[Any]:
        return [
            Header(text="Hello World"),
            Input(
                key="template",
                text="Greeting template",
                default=DEFAULT_TEMPLATE,
                subtext="Use {name} where the entered name should appear.",
                icon="msg_edit",
            ),
            Text(
                text="Example",
                subtext=".hello Alice -> Hello, Alice!",
                icon="msg_info",
            ),
        ]
 
    def on_send_message_hook(self, account: int, params: Any) -> HookResult:
        # Some sends are media-only, so always validate that we really have text.
        if not isinstance(getattr(params, "message", None), str):
            return HookResult()
 
        raw_text = params.message.strip()
        if not raw_text.startswith(".hello"):
            return HookResult()
 
        # Split only once so ".hello Alice Smith" keeps the full name.
        parts = raw_text.split(" ", 1)
        name = parts[1].strip() if len(parts) > 1 else ""
 
        if not name:
            params.message = "Usage: .hello <name>"
            return HookResult(strategy=HookStrategy.MODIFY, params=params)
 
        # Read the saved template or fall back to the default one.
        template = self.get_setting("template", DEFAULT_TEMPLATE)
 
        try:
            params.message = template.format(name=name)
        except Exception:
            # If the user saved a broken template, fail softly and explain why.
            params.message = "Template must contain valid Python format placeholders, for example {name}."
 
        # Tell the plugin engine that we changed outgoing message params.
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
```

## What this example teaches

### 1\. Metadata is parsed statically

The loader reads metadata from the Python file with `AST`, so values such as `__id__` and `__version__` should stay as plain top-level constants.

### 2\. Hooks must be registered

Implementing `on_send_message_hook(...)` is not enough by itself. You also need:

```python
def on_plugin_load(self):
    self.add_on_send_message_hook()
```

### 3\. Settings are regular Python dataclasses

The settings page is built from objects from `ui.settings`.

In this example:

  * `Header` creates a section title
  * `Input` persists a string under the key `"template"`
  * `Text` is just a read-only helper row

### 4\. `HookResult` decides what happens next

For outgoing messages:

  * `HookResult()` means "leave it unchanged"
  * `HookResult(strategy=HookStrategy.MODIFY, params=params)` means "use the updated params"
  * `HookResult(strategy=HookStrategy.CANCEL)` means "stop the send entirely"

## Testing it

  1. Enable the plugin.
  2. Open the plugin settings and change the template if you want.
  3. Send `.hello Alice` in any chat.

Expected result:

```python
Hello, Alice!
```

If you change the template to:

```python
Welcome back, {name}.
```

then `.hello Alice` becomes:

```python
Welcome back, Alice.
```

## Common beginner mistakes

  * forgetting `self.add_on_send_message_hook()` in `on_plugin_load`
  * building metadata dynamically instead of using plain constants
  * assuming `params.message` always exists
  * doing slow network or file work directly in a hook on the UI thread

If your plugin needs background work, continue with [Client Utilities](</docs/client-utils>), especially `run_on_queue(...)`.

## What to explore next

After this plugin, the most useful next steps are:

  * [Plugin Class](</docs/plugin-class>) for lifecycle and hooks
  * [Plugin Settings](</docs/plugin-settings>) for richer settings UIs
  * [Client Utilities](</docs/client-utils>) for sending messages, requests, and queue work

To further develop your plugin skills, explore community-made plugins:

### [exteraGram PluginsOfficial Channel](<https://t.me/exteraPlugins>)### [exteraGram PluginsPlugins from supporters; maintained by [@itsv1eds](<https://t.me/itsv1eds>)](<https://t.me/exteraPluginsSup>)### [DottedPlugins by [@exteraDev](<https://t.me/exteraDev>)](<https://t.me/exteraDevPlugins>)### [zwyPluginsPlugins by [@zwylair](<https://t.me/zwylair>)](<https://t.me/zwyPlugins>)

[PreviousSetup](</docs/setup>)[NextElyx](</docs/elyx>)


---


# === DOCUMENT: Plugin Class (/docs/plugin-class) ===

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


---


# === DOCUMENT: Multi-Account (/docs/multi-account) ===

---
title: "Multi-Account"
source: "https://plugins.exteragram.app/docs/multi-account"
---

# Multi-account

How account scoping works, why hooks fire for accounts that are not on screen, and how to target the right one.

exteraGram keeps every logged-in account connected at the same time. Accounts that are not on screen still receive updates, still send requests, and still fire your hooks.

That matters because the account helpers in `client_utils` default to **the account selected in the UI** , which is global state that has nothing to do with the account your callback was invoked for. Reply from inside a hook without thinking about it and the message can go out from the wrong account.

## The rule

Every hook already tells you which account it is for:

```python
def on_update_hook(self, update_name: str, account: int, update: Any) -> HookResult:
```

Use that `account` for anything you do in response to it.

## Targeting an account

Every account-scoped helper takes an optional `account`:

```python
from client_utils import get_messages_controller, get_user_config, send_text
 
get_messages_controller(account)
get_user_config(account)
send_text(peer, "hello", account=account)
```

When you do more than one thing, take a client instead of repeating yourself. `self.client(...)` returns an `AccountClient` with the same method names, bound to one account:

```python
from base_plugin import BasePlugin, HookResult
 
 
class ReplyPlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_hook("TL_updateNewMessage")
 
    def on_update_hook(self, update_name: str, account: int, update) -> HookResult:
        if update_name != "TL_updateNewMessage":
            return HookResult()
 
        client = self.client(account)
 
        me = client.get_user_config().getCurrentUser()
        client.send_text(update.message.dialog_id, f"replying as {me.first_name}")
 
        return HookResult()
```

`get_client(account)` in `client_utils` does the same thing outside of a plugin class.

## What happens if you forget

Nothing changes silently. A helper called without an `account` still uses the selected account, exactly as before — but if it runs while you are handling a different account, the SDK logs a warning once per helper:

```python
get_messages_controller() is using account 0 because it is the one selected in the UI,
but it was called while handling account 1. Pass account=1, or use get_client(1),
if you meant that account. Logged once per helper.
```

Seeing that in your logs means the plugin is reading or writing the wrong account.

The scope follows your work: it is set around every hook callback, around `run_on_queue` work scheduled from one, and around `send_request` callbacks. `get_hook_account()` returns it, or `None` outside a callback, and `self.client()` with no argument follows it.

## Deliberately using the selected account

Sometimes the selected account is genuinely what you want, for example writing to the Saved Messages of whoever is looking at the screen. Say so explicitly and the warning goes away:

```python
from client_utils import get_selected_account, send_text
 
send_text(peer, "note to self", account=get_selected_account())
```

## Notification delegates

`NotificationCenterDelegate.didReceivedNotification` receives an `account` too, and it is not scoped automatically. Take a client from it yourself:

```python
class MyDelegate(NotificationCenterDelegate):
    def didReceivedNotification(self, id: int, account: int, args):
        get_client(account).get_messages_controller()
```

## Requirements

The `account` parameters, `AccountClient`, `get_client()`, `get_hook_account()` and `get_selected_account()` need SDK 1.4.5.0 or newer. Declare it so older installs fail with a clear message instead of a `TypeError`:

```python
__sdk_version__ = ">=1.4.5.0"
```

[PreviousPlugin Class](</docs/plugin-class>)[NextPlugin Settings](</docs/plugin-settings>)


---


# === DOCUMENT: Plugin Settings (/docs/plugin-settings) ===

---
title: "Plugin Settings"
source: "https://plugins.exteragram.app/docs/plugin-settings"
---

# Plugin Settings

Learn how to build settings pages, sub-pages, and custom setting items for your plugin.

You can build a plugin settings screen by implementing `create_settings(self) -> List[Any]` on your `BasePlugin`.

The method should return a list of dataclass instances from `ui.settings`.

## General Example

This example intentionally uses almost every built-in setting type so you can see how the pieces fit together.

```python
from typing import Any, List
 
from android.view import Gravity, View
from android.widget import LinearLayout
 
from org.telegram.messenger import AndroidUtilities, MediaDataController, UserConfig
from org.telegram.ui.Components import BackupImageView, LayoutHelper, UItem
 
from base_plugin import BasePlugin
from ui.settings import (
    Custom,
    Divider,
    EditText,
    Header,
    Input,
    Selector,
    SimpleSettingFactory,
    Switch,
    Text,
)
 
 
def create_sticker_view(context, list_view, current_account: int, class_guid: int, resources_provider):
    main_layout = LinearLayout(context)
    main_layout.setOrientation(LinearLayout.VERTICAL)
    main_layout.setGravity(Gravity.CENTER)
    main_layout.setPadding(
        AndroidUtilities.dp(20),
        AndroidUtilities.dp(20),
        AndroidUtilities.dp(20),
        AndroidUtilities.dp(20),
    )
 
    image_view = BackupImageView(context)
    image_view.setRoundRadius(AndroidUtilities.dp(45))
 
    # The view is created here once. Do not bind item-specific data here yet.
    main_layout.addView(image_view, LayoutHelper.createLinear(130, 130, Gravity.CENTER, 0, 0, 0, 16))
    return main_layout
 
 
def bind_sticker_view(view, item, divider: bool, adapter, list_view):
    sticker_set_name = "CactusPlugins"
    sticker_index = 0
 
    # This runs when the view should display actual content.
    MediaDataController.getInstance(UserConfig.selectedAccount).setPlaceholderImageByIndex(
        view.getChildAt(0),
        sticker_set_name,
        sticker_index,
        "130_130",
    )
 
 
StickerSettingFactory = SimpleSettingFactory(
    create_sticker_view,
    bind_sticker_view,
    is_clickable=False,
    is_shadow=False,
)
 
 
class MyPlugin(BasePlugin):
    def _on_test_switch_change(self, new_value: bool):
        self.logger.info(f"Test switch changed to: {new_value}")
 
    def _on_test_input_change(self, new_value: str):
        self.logger.info(f"Test input changed to: {new_value}")
 
    def _on_test_selector_change(self, new_index: int):
        self.logger.info(f"Test selector changed to index: {new_index}")
 
    def _on_text_click(self, view: View):
        self.logger.info("Text item clicked!")
 
    def _create_sub_page(self) -> List[Any]:
        return [
            Header(text="This is a Sub-Page"),
            Text(text="You can nest settings pages."),
        ]
 
    def create_settings(self) -> List[Any]:
        return [
            Header(text="General Settings"),
            Switch(
                key="test_switch_key",
                text="Test Switch",
                default=True,
                subtext="This is a sample switch control.",
                icon="msg_settings",
                on_change=self._on_test_switch_change,
                link_alias="test_switch",
            ),
            Selector(
                key="test_selector_key",
                text="Test Selector",
                default=1,
                items=["Option A", "Option B", "Option C"],
                icon="msg_list",
                on_change=self._on_test_selector_change,
            ),
            Input(
                key="test_input_key",
                text="Test Input",
                default="Hello, World!",
                subtext="A simple single-line text input.",
                icon="msg_text",
                on_change=self._on_test_input_change,
            ),
            EditText(
                key="multiline_key",
                hint="Enter multiple lines of text here...",
                default="",
                multiline=True,
                max_length=1000,
            ),
            Divider(text="This is a divider with text."),
            Text(
                text="Click for Sub-Page",
                icon="msg_arrow_forward",
                on_click=self._on_text_click,
                create_sub_fragment=self._create_sub_page,
                link_alias="sub_page_link",
            ),
            Text(
                text="This is red text",
                icon="msg_error",
                red=True,
            ),
            Divider(),
            Header(text="Custom Controls"),
            Custom(factory=StickerSettingFactory.instance.java),
            Custom(item=UItem.asShadow("This is a custom UItem shadow.")),
        ]
```

## Accessing and Modifying Settings

Use the helper methods from `BasePlugin`.

```python
# Get the value of "test_switch_key", defaulting to False if it was never saved.
is_enabled = self.get_setting("test_switch_key", False)
```

To save a value:

```python
# Example: toggle a boolean setting.
current_value = self.get_setting("test_switch_key", False)
self.set_setting("test_switch_key", not current_value)
 
# If changing one setting should rebuild the whole page,
# ask the plugin controller to reload the settings screen.
self.set_setting("main_option", "A", reload_settings=True)
```

You can also export or import all settings for the current plugin:

```python
# Export all settings for backup or debugging.
all_my_settings = self.export_settings()
self.logger.info(f"My settings: {all_my_settings}")
 
# Import a whole dictionary of settings.
new_settings = {
    "test_switch_key": False,
    "test_input_key": "New Value",
}
 
# By default the settings UI is reloaded after import.
self.import_settings(new_settings)
 
# You can skip reload when doing batch work.
self.import_settings(new_settings, reload_settings=False)
```

## Built-in Setting Types

The SDK currently exposes these public dataclasses from `ui.settings`:

  * `Header`
  * `Divider`
  * `Switch`
  * `Selector`
  * `Input`
  * `Text`
  * `EditText`
  * `Custom`

### Supported Controls

Control| `key`| `text`| `default`| Other Important Parameters  
---|---|---|---|---  
**Header**|  -| **Required**|  -| `text`: section title  
**Divider**|  -| -| -| `text`: optional caption shown on the divider  
**Switch**| **Required**| **Required**| **Required** (`bool`)| `subtext`, `icon`, `on_change`, `on_long_click`, `link_alias`  
**Selector**| **Required**| **Required**| **Required** (`int`)| `items`, `icon`, `on_change`, `on_long_click`, `link_alias`  
**Input**| **Required**| **Required**|  optional `str`| `subtext`, `icon`, `on_change`, `on_long_click`, `link_alias`  
**Text**|  -| **Required**|  -| `subtext`, `icon`, `accent`, `red`, `on_click`, `on_long_click`, `create_sub_fragment`, `link_alias`  
**EditText**| **Required**|  -| optional `str`| `hint` (required), `multiline`, `max_length`, `mask`, `on_change`  
**Custom**|  -| -| -| `item`, `view`, `factory`, `factory_args`, `on_click`, `on_long_click`, `create_sub_fragment`, `link_alias`  
  
### Parameter Details

Parameter| Type| Description  
---|---|---  
`key`| `str`| Unique key for persisted settings. Used with `get_setting()` and `set_setting()`.  
`text`| `str`| Main visible text for the setting row.  
`default`| `Any`| Default value used if the setting was never saved.  
`subtext`| `str`| Optional second line of text.  
`icon`| `str`| Optional drawable name such as `"msg_settings"`.  
`on_change`| `Callable`| Runs when the value changes. Signature depends on control type.  
`on_click`| `Callable`| Runs when the row is clicked. Usually receives the clicked `View`.  
`on_long_click`| `Callable`| Runs on long press. Usually receives the clicked `View`.  
`link_alias`| `str`| Enables a copyable deeplink alias for the setting row.  
`items`| `List[str]`| Selector option labels.  
`create_sub_fragment`| `Callable[[], List[Any]]`| Builds a nested settings page when the item is tapped.  
`accent`| `bool`| Applies accent coloring to `Text`.  
`red`| `bool`| Applies red coloring to `Text`.  
`hint`| `str`| Placeholder text for `EditText`.  
`multiline`| `bool`| Allows multiple lines in `EditText`.  
`max_length`| `int`| Maximum allowed input length.  
`mask`| `str`| Regex-like input restriction for `EditText`.  
`factory`| `CustomSetting.Factory`| Java factory used for advanced custom items.  
`item`| `UItem`| Prebuilt Telegram `UItem` instance.  
`view`| `View`| Plain Android `View` instance.  
  
## Custom Settings

`Custom` is the flexible entry point for settings rows which do not fit the built-in controls.

There are three main ways to use it.

### 1\. Provide an existing `UItem`

This is the lightest solution when Telegram already has a suitable row type.

```python
from org.telegram.ui.Components import UItem
from ui.settings import Custom
 
 
setting = Custom(
    item=UItem.asShadow("This is a custom UItem shadow.")
)
```

### 2\. Provide a ready Android `View`

This is useful for static or already-constructed views.

```python
from android.widget import TextView
from ui.settings import Custom
 
 
label = TextView(context)
label.setText("Hello from a custom View")
 
setting = Custom(view=label)
```

This form is usually best for simple one-off views. If the row needs recycling, binding, click handling, or per-item behavior, prefer a factory.

### 3\. Use `SimpleSettingFactory`

`SimpleSettingFactory` is the recommended high-level API for custom settings.

It internally creates a Java `CustomSetting.Factory`, handles binding, and exposes Python callbacks for the parts you usually want to customize.

```python
from android.view import Gravity
from android.widget import LinearLayout, TextView
 
from org.telegram.messenger import AndroidUtilities
from org.telegram.ui.Components import LayoutHelper
 
from ui.bulletin import BulletinHelper
from ui.settings import Custom, SimpleSettingFactory
 
 
def create_view(context, list_view, current_account: int, class_guid: int, resources_provider):
    main_layout = LinearLayout(context)
    main_layout.setOrientation(LinearLayout.VERTICAL)
    main_layout.setGravity(Gravity.CENTER)
    main_layout.setPadding(
        AndroidUtilities.dp(20),
        AndroidUtilities.dp(20),
        AndroidUtilities.dp(20),
        AndroidUtilities.dp(20),
    )
 
    # Build the row skeleton once.
    title = TextView(context)
    title.setText("Tap me")
    main_layout.addView(title, LayoutHelper.createLinear(LayoutHelper.WRAP_CONTENT, LayoutHelper.WRAP_CONTENT))
    return main_layout
 
 
def bind_view(view, item, divider: bool, adapter, list_view):
    # Bind current data here.
    title = view.getChildAt(0)
    title.setText("Tap me")
 
 
def on_click(plugin, item, view):
    BulletinHelper.show_info(f"Hello from {plugin.getName()}!")
 
 
InteractiveRowFactory = SimpleSettingFactory(
    create_view,
    bind_view,
    is_clickable=True,
    on_click=on_click,
)
 
 
setting = Custom(factory=InteractiveRowFactory.instance.java)
```

### Factory callbacks

`SimpleSettingFactory(...)` supports these callbacks:

  * `create_view(context, list_view, current_account, class_guid, resources_provider) -> View`
  * `bind_view(view, item, divider, adapter, list_view) -> None`
  * `create_item(plugin, setting, args) -> UItem`
  * `on_click(plugin, item, view) -> None`
  * `on_long_click(plugin, item, view) -> bool`
  * `attached_view(list_view, view, item) -> None`
  * `equals(a, b) -> bool`
  * `content_equals(a, b) -> bool`

Constructor options:

```python
SimpleSettingFactory(
    create_view,
    bind_view,
    is_clickable=False,
    is_shadow=False,
    create_item=None,
    on_click=None,
    on_long_click=None,
    attached_view=None,
    equals=None,
    content_equals=None,
)
```

A few important notes:

  * `create_view(...)` builds the row view itself.
  * `bind_view(...)` fills the current data into that view.
  * `is_clickable` and `is_shadow` are optimized Java-side flags used in hot paths.
  * if `create_item` is omitted, the factory creates a default `UItem` for you
  * if `equals` or `content_equals` are omitted, Telegram's default `UItem` comparison logic is used

### Using `SimpleSettingFactory` as a callable

`SimpleSettingFactory` also implements `__call__`, so you can use it as a small builder for `Custom(...)`.

```python
StickerFactory = SimpleSettingFactory(create_sticker_view, bind_sticker_view)
 
 
setting = StickerFactory(link_alias="sticker_preview")
```

You can also pass positional arguments:

```python
setting = StickerFactory("CactusPlugins", 0)
```

Those values are stored in `factory_args` for the created custom row and can be used by advanced factory logic.

### Factory lifetime

Create each factory once, for example at module level, and reuse it whenever `create_settings()` runs. A factory stays registered, together with the functions passed to it, until the plugin is unloaded, so creating new ones each time the settings are built makes memory use grow while the plugin runs.

If a plugin does need temporary factories, it can unregister one with `release()` once no settings page shows it any more:

```python
factory = SimpleSettingFactory(create_view, bind_view)
...
factory.release()
```

## Advanced Custom Factories

If `SimpleSettingFactory` is not enough, you can generate your own `CustomSetting.Factory` subclass through the class proxy system.

That route is useful when you need:

  * a real Java subclass
  * custom Java fields
  * `super()` support
  * overload-specific behavior
  * MVEL-backed hot-path methods

For that API, see [Class Proxy](</docs/class-proxy>).

[PreviousMulti-account](</docs/multi-account>)[NextLogging](</docs/logging>)


---


# === DOCUMENT: Logging (/docs/logging) ===

---
title: "Logging"
source: "https://plugins.exteragram.app/docs/logging"
---

# Logging

Plugin logger with levels, per-plugin logcat tags, tracebacks, object dumps, timers and handlers.

The `log_utils` module is the SDK's logger. Every plugin gets one as `self.logger`; it writes to logcat under the tag `plugin.<your plugin id>`, so you can filter your plugin's output without grepping.

```python
from base_plugin import BasePlugin
 
 
class MyPlugin(BasePlugin):
    def on_plugin_load(self):
        self.logger.info("Loaded version %s", self.version)
        self.logger.debug("Settings: %s", self.export_settings())
```

Read it with:

```python
adb logcat -s plugin.my_plugin
```

Every entry starts with the place it was logged from:

```python
I plugin.my_plugin: [my_plugin.py:on_plugin_load:6] Loaded version 1.0
```

> > [!NOTE] **Replaces log() and self.log()** > android_utils.log()andBasePlugin.log()still work, but they are legacy: they always log at debug level under a tag you can't filter by. SeeMigrating from log().

## Levels

Level| Method| Use for  
---|---|---  
`LogLevel.VERBOSE`| `logger.verbose(...)`| very chatty tracing, e.g. every hook call  
`LogLevel.DEBUG`| `logger.debug(...)`| details you need while developing  
`LogLevel.INFO`| `logger.info(...)`| normal lifecycle events  
`LogLevel.WARNING`| `logger.warning(...)`| something unexpected that the plugin survived  
`LogLevel.ERROR`| `logger.error(...)`, `logger.exception(...)`| failures  
  
Levels map to Android log priorities, so `adb logcat *:E` shows only errors.

By default a logger shows **DEBUG and above while developer mode is on** (the switch on the plugins info screen, see [Setup](</docs/setup>)) and **INFO and above otherwise**. Users of your released plugin don't pay for debug logs, and you see them while developing.

Change it per logger:

```python
from log_utils import LogLevel
 
self.logger.level = LogLevel.VERBOSE  # also accepts "verbose" or 2
self.logger.level = LogLevel.OFF      # silence the logger
self.logger.level = None              # back to the default
```

> > [!NOTE] **Pass arguments instead of f-strings** > self.logger.debug("chat %s: %s", chat_id, message)only formats the message when DEBUG is enabled. An f-string is built every time, even if nothing gets logged. This matters in hooks that run thousands of times.

## Errors

Call `exception()` inside an `except` block, and the traceback is added for you:

```python
try:
    self.process(update)
except Exception:
    self.logger.exception("Failed to process %s", update_name)
```

Outside an `except` block, pass the exception explicitly:

```python
self.logger.error("Request failed", exc_info=error)
```

## Inspecting objects

`dump()` prints the structure of an object: Telegram `TLRPC` objects, other Java objects, and Python values alike.

```python
self.logger.dump(user, "user")              # logs at DEBUG
self.logger.dump(message.messageOwner, depth=3)
```

```python
user: TLRPC.TL_user @3f2a1b
  ...
  first_name = 'Ivan'
  id = 123456789
  ...
  photo = TLRPC.TL_userProfilePhoto @9c01de
    dc_id = 2
    ...
    photo_id = 5221840139847234
    stripped_thumb = byte[412]
  ...
```

How values are shown:

  * Only fields are read. Getters, Python properties and `__getattr__` never run, so dumping an object has no side effects.
  * Java objects are walked through their class hierarchy, without static fields. Classes from `java.*`, `android.*` and similar platform packages are shown with `toString()` instead.
  * Lists, maps, arrays and sparse arrays show their size and the first `max_items` entries (default 20).
  * Nesting stops at `depth` levels (default 2) and is marked with `{…}`. An object that contains itself is marked `<cycle>`.

`dump()` is also a plain function that returns a string, so you can use it outside logging:

```python
from android_utils import copy_to_clipboard
from log_utils import dump
 
copy_to_clipboard(dump(message.messageOwner, depth=4))
```

## Child loggers

Split a large plugin into areas with their own level:

```python
self.net_log = self.logger.child("net")    # tag: plugin.my_plugin.net
self.net_log.level = LogLevel.VERBOSE
```

A child uses its parent's level until you set its own.

In helper modules, get a logger without passing the plugin around:

```python
from log_utils import get_logger
 
logger = get_logger(__name__)
```

`get_logger()` finds which plugin is calling it from the file it is called in. In the plugin's main file it returns the same logger as `self.logger`, and in Elyx plugin modules it returns a child named after the module.

## Measuring time

```python
with self.logger.timed("Load stickers"):
    self.load_stickers()
# D plugin.my_plugin: [my_plugin.py:on_plugin_load:21] Load stickers took 12.4 ms
```

If the block raises, the entry ends with `(failed)` and the exception keeps propagating.

## Handlers

Handlers receive every entry of a logger and of its children, after level filtering:

```python
from android_utils import run_on_ui_thread
from log_utils import LogLevel, LogRecord
from ui.bulletin import BulletinHelper
 
 
class MyPlugin(BasePlugin):
    def on_plugin_load(self):
        self.logger.add_handler(self.show_errors)
 
    def show_errors(self, record: LogRecord):
        if record.level >= LogLevel.ERROR:
            run_on_ui_thread(lambda: BulletinHelper.show_error(record.message))
```

`LogRecord` fields: `level`, `tag`, `message`, `location` (`file:function:line` or `None`), `traceback` (`str` or `None`), `created` (Unix time), and `text`, which is the entry exactly as written to logcat.

  * Handlers run synchronously on the thread that logged, so keep them fast.
  * Logging from inside a handler doesn't call the handlers again.
  * An exception in a handler is logged under the `pysdk` tag and doesn't affect your plugin.
  * When the plugin is unloaded, the SDK removes its handlers and resets its levels.

## Helper functions and `stacklevel`

If you wrap the logger in your own helper, pass `stacklevel=2` so the location points at the helper's caller rather than the helper:

```python
def warn_once(self, key, text):
    if key not in self.warned:
        self.warned.add(key)
        self.logger.warning(text, stacklevel=2)
```

## Long messages

Logcat cuts a single entry at about 4 KB. The logger splits longer messages, such as big dumps or tracebacks, into several entries at line boundaries, so nothing gets lost.

## Migrating from `log()`

Before| After  
---|---  
`self.log("Loaded")`| `self.logger.info("Loaded")`  
`log(f"Value: {x}")`| `logger.debug("Value: %s", x)`  
`log(user)`| `logger.dump(user)`  
`log(f"Error: {e}\n{traceback.format_exc()}")`| `logger.exception("Error")`  
  
The old functions keep working. `log(obj)` with a non-text value now prints the same output as `dump()`.

If your plugin already assigns its own `self.logger`, that keeps working too: your value is returned instead of the SDK logger.

## Reference

```python
class Logger:
    tag: str
    level: LogLevel | None            # None: use the parent's or the default level
    effective_level: LogLevel
 
    def verbose(msg, *args, exc_info=None, stacklevel=1)
    def debug(msg, *args, exc_info=None, stacklevel=1)
    def info(msg, *args, exc_info=None, stacklevel=1)
    def warning(msg, *args, exc_info=None, stacklevel=1)
    def error(msg, *args, exc_info=None, stacklevel=1)
    def exception(msg, *args, stacklevel=1)
    def log(level, msg, *args, exc_info=None, stacklevel=1)
    def dump(obj, label=None, *, level=LogLevel.DEBUG, depth=2, max_items=20, stacklevel=1)
    def timed(label, level=LogLevel.DEBUG)   # context manager
    def is_enabled_for(level) -> bool
    def child(name) -> Logger
    def add_handler(handler)
    def remove_handler(handler) -> bool
 
def get_logger(name: str | None = None) -> Logger
def dump(obj, depth=2, max_items=20) -> str
```

`Logger(tag)` creates a standalone logger with any tag, if you need one outside the plugin hierarchy.

[PreviousPlugin Settings](</docs/plugin-settings>)[NextClass Proxy](</docs/class-proxy>)


---


# === DOCUMENT: Class Proxy (/docs/class-proxy) ===

---
title: "Class Proxy"
source: "https://plugins.exteragram.app/docs/class-proxy"
---

# Class Proxy

Create Java subclasses and dynamic proxy classes from Python.

# Class Proxy

The class proxy API allows a plugin to create a real Java class at runtime and implement it in Python.

This is useful when a Java API expects:

  * a subclass of an existing Java class
  * an implementation which overrides parent methods
  * extra methods declared directly on the generated Java class
  * Java fields stored on the generated object

The proxy system is built on top of `DexMaker`, so the result is an actual Java class which can be passed anywhere the app expects that type.

`new_instance()` returns a Python peer object, not the raw Java instance. Use `.java` when an API explicitly requires the generated Java object itself.

## Recommended API

The recommended API is the managed DSL from `extera_utils.classes`.

```python
from extera_utils.classes import (
    Base,
    java_subclass,
    joverride,
    joverload,
    jmethod,
    jMVELmethod,
    jMVELoverride,
    jclassbuilder,
    jfield,
    jgetmethod,
    jsetmethod,
    jconstructor,
    jpreconstructor,
)
```

Main pieces:

  * `Base`: base Python class for managed Java subclasses
  * `@java_subclass(JavaClass, Interface1, Interface2, ..., methods=..., constructors=..., custom_name=...)`: binds your Python class to a Java base class and optional Java interfaces
  * `@joverride(...)`: overrides an existing Java method
  * `@joverload(name, arg_types)`: overload-friendly alias for `@joverride`
  * `@jmethod(...)`: adds a new method directly to the generated Java class
  * `jMVELmethod(...)`: adds a Java method backed by MVEL instead of Python
  * `jMVELoverride(...)`: overrides a parent Java method with MVEL code
  * `@jclassbuilder()`: lets you modify `DexMaker` before the class is finalized and loaded
  * `jfield(...)`: adds a Java field to the generated class
  * `jgetmethod(...)` / `jsetmethod(...)`: generate Java-only getter/setter methods for a field
  * `@jconstructor(...)`: runs Python code after the Java constructor has finished
  * `@jpreconstructor(...)`: modifies constructor arguments before calling the Java parent constructor

Useful aliases also exist:

  * `jmvelmethod` / `jMVELmethod`
  * `jmveloverride` / `jMVELoverride`
  * `joverride`, `joverload`, `jmethod`, `jfield`, `jconstructor`, `jpreconstructor`
  * `jgetmethod`, `jsetmethod`, `jclassbuilder`

## Quick Start

```python
from extera_utils.classes import Base, java_subclass, joverload, jfield
from java.util import ArrayList
 
 
@java_subclass(ArrayList)
class CountingList(Base):
    added_count = jfield("int", default=0)
 
    @joverload("add", ["java.lang.Object"])
    def add_item(self, value):
        self.added_count += 1
        return super().add_item(value)
 
 
items = CountingList.new_instance()
items.add("Hello")
items.add("world")
```

What happens here:

  * `CountingList` becomes a generated Java subclass of `java.util.ArrayList`
  * `added_count` becomes a real Java field on that class
  * `add(Object)` is overridden in Java and routed into Python
  * `super().add_item(...)` calls the original parent Java implementation

Because `ArrayList.add` is overloaded, the example uses `@joverload(...)`. For overloaded Java methods, prefer explicit signatures instead of bare `@joverride()`.

## Creating a Java Subclass

Use `@java_subclass` for the most readable syntax:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    ...
```

If the generated Java class must also implement interfaces, pass them after the base class:

```python
@java_subclass(FrameLayout, CustomDelegate1, CustomDelegate2)
class MyCell(Base):
    ...
```

This produces a generated Java class equivalent to:

```python
class GeneratedProxy extends FrameLayout implements CustomDelegate1, CustomDelegate2
```

### Custom generated class name

> > [!NOTE] **App version** > Work only in app, which version code >=66690

You can control the middle part of the generated Java proxy class name with `custom_name=...`.

```python
@java_subclass(Object, custom_name="DialogCell")
class Test(Base):
    ...
```

That generates a class name in this shape:

```python
Proxy_Object_DialogCell_<hashcode>_<randomnum>
```

If `custom_name` is not provided, the proxy system uses the Python class name automatically:

```python
@java_subclass(Object)
class Test(Base):
    ...
```

This becomes:

```python
Proxy_Object_Test_<number>_<number>
```

The same option is available on `bind(...)`:

```python
MySubclass.bind(SomeJavaClass, custom_name="SettingsRow")
```

Names are sanitized before generation, so unsupported characters are replaced with `_`.

Generated proxy classes are also loaded through a shared generated-class loader chain, so a proxy created later can reference earlier generated proxy classes when needed.

This is still not a substitute for a stable API type. If possible, prefer referencing another proxy through an interface or a common base class instead of depending on the exact generated class.

You can also bind after the class definition:

```python
class MySubclass(Base):
    ...
 
 
MySubclass.bind(SomeJavaClass)
```

And `bind(...)` supports the same interface list:

```python
MySubclass.bind(FrameLayout, CustomDelegate1, CustomDelegate2)
```

To get the generated Java class:

```python
proxy_java_class = MySubclass.java_class()
```

To create an instance:

```python
instance = MySubclass.new_instance()
instance2 = MySubclass.new_instance("text", 123)  # provide java-constructor args
```

`instance` is a Python peer of `MySubclass`. If you need the raw Java object:

```python
java_instance = instance.java
java_instance2 = MySubclass.new_java_instance("text", 123)
```

If you want to pass separate arguments into the Python `__init__` method, use `init_args`:

```toml
instance = MySubclass.new_instance("java ctor value", init_args=["python init value"])
```

In this case:

  * `"java ctor value"` is used for Java constructor resolution
  * `["python init value"]` is passed to Python `__init__`

If you already have a raw Java instance of the generated class and want to recover its Python peer, use `from_java(...)`:

```python
existing_java = MySubclass.new_java_instance("text")
peer = MySubclass.from_java(existing_java)
```

This is useful when:

  * some Java API created or stored the proxy instance for you
  * you received the generated object back through another callback
  * you want to continue working with the Python peer instead of the raw Java object

## Overriding Methods

### Simple Override

If the Java method name is the same as the Python method name:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride()
    def onClick(self, view):
        print("Clicked")
```

### Override With Another Python Name

If the Java method name is not convenient in Python, pass it explicitly:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride("equals")
    def equals_(self, a, b):
        return a == b
```

This is useful for names such as `equals`, `hashCode`, or when you want several Python handlers for overloaded methods.

## Overloaded Methods

When a Java class has several methods with the same name, use the overload-aware decorators.

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverload("setValue", ["int"])
    def set_value_int(self, value):
        print("int overload", value)
 
    @joverload("setValue", ["java.lang.String"])
    def set_value_text(self, value):
        print("string overload", value)
```

You can write the same thing with `joverride`:

```python
@joverride("setValue", ["int"])
def set_value_int(self, value):
    ...
```

### Supported Type Formats

Argument and return types may be specified as:

  * primitive names like `"int"`, `"boolean"`, `"long"`
  * fully-qualified class names like `"java.lang.String"`
  * Java classes returned by `jclass(...)`
  * Java primitive type objects

Example:

```python
from java import jclass
 
View = jclass("android.view.View")
 
@joverload("setView", [View])
def set_view(self, view):
    ...
```

## Calling Parent Methods With `super()`

For overridden methods, `super()` works like a normal Python method call:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride()
    def onClick(self, view):
        print("Before parent")
        result = super().onClick(view)
        print("After parent")
        return result
```

For overloaded methods, call the Python method name you defined:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverload("setValue", ["int"])
    def set_value_int(self, value):
        return super().set_value_int(value)
```

## Adding New Java Methods

Use `@jmethod` to declare a method which does not exist on the parent Java class.

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @jmethod("debugLabel", "java.lang.String", ["int"])
    def build_debug_label(self, value):
        return f"value={value}"
```

This creates a real Java method named `debugLabel(int)` on the generated class.

### `jmethod` Signatures

These forms are supported:

```python
@jmethod("java.lang.String", ["int"])
def my_method(self, value):
    ...
 
@jmethod("debugLabel", "java.lang.String", ["int"])
def build_debug_label(self, value):
    ...
```

If the return type and argument types are not passed explicitly, `jmethod` can infer them from Python annotations:

```python
@jmethod()
def build_debug_label(self, value: "int") -> "java.lang.String":
    return f"value={value}"
```

Explicit types are still recommended when the Java signature matters a lot.

## Adding MVEL Methods

If a method is called very often and the logic is simple enough to keep on the Java side, you can generate a method whose body is executed by `MVEL` instead of calling Python.

```python
from extera_utils.classes import Base, java_subclass, jMVELmethod
 
 
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    debug_label = jMVELmethod(
        return_type="java.lang.String",
        arguments=[("prefix", "java.lang.String"), ("value", "int")],
        code="""
            return prefix + ": value=" + value + ", class=" + java.getClass().getSimpleName();
        """,
    )
```

This creates a real Java method named `debug_label(String, int)` on the generated class, but its implementation runs through compiled MVEL.

### Overriding With MVEL

For parent methods, use `jMVELoverride(...)`:

```python
from extera_utils.classes import Base, java_subclass, jMVELoverride
 
 
@java_subclass(SomeFactoryClass)
class OptimizedFactory(Base):
    isClickable = jMVELoverride(
        arguments=[],
        code="return factory_is_clickable;",
    )
```

If the parent method is overloaded, pass explicit argument types just like `@joverload(...)`:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    add = jMVELoverride(
        arguments=[("value", "java.lang.Object")],
        code="""
            add_calls = add_calls + 1;
            return SUPER_add(value);
        """,
    )
```

### MVEL Context

Inside MVEL:

  * the root object is the generated Java instance, so Java fields and methods can be accessed directly
  * `java`: the same generated Java instance
  * `python` / `py` / `self`: the attached Python peer object
  * `args`: `Object[]` with all method arguments
  * `argc`: number of arguments
  * `arg0`, `arg1`, ...: positional arguments
  * named arguments from `arguments=[("name", "type"), ...]`

That means all of these are valid:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    describe = jMVELmethod(
        return_type="java.lang.String",
        arguments=[("value", "int")],
        code="""
            return "value=" + value + ", class=" + java.getClass().getSimpleName();
        """,
    )
```

For override methods, the generated `SUPER_<methodName>(...)` bridge is also available:

```python
@java_subclass(SomeViewClass)
class MyView(Base):
    onMeasure = jMVELoverride(
        arguments=[("widthMeasureSpec", "int"), ("heightMeasureSpec", "int")],
        code="""
            SUPER_onMeasure(
                android.view.View$MeasureSpec.makeMeasureSpec(
                    android.view.View$MeasureSpec.getSize(widthMeasureSpec),
                    android.view.View$MeasureSpec.EXACTLY
                ),
                android.view.View$MeasureSpec.makeMeasureSpec(
                    org.telegram.messenger.AndroidUtilities.dp(68),
                    android.view.View$MeasureSpec.EXACTLY
                )
            );
            return null;
        """,
    )
```

That pattern is useful for small hot-path UI overrides where the method body is simple but called very often.

### When To Use MVEL

  * Use `jfield(..., methods=[jgetmethod(...), jsetmethod(...)])` for pure field getter/setter hot paths. That is still the fastest option.
  * Use `jMVELmethod(...)` or `jMVELoverride(...)` when the method needs a bit of logic, but you still want to avoid calling into Python every time.
  * Keep MVEL bodies small and predictable. If the logic needs Python objects, complex state, or heavy control flow, regular Python overrides are usually easier to maintain.

## Adding Java Fields

Use `jfield` to declare fields on the generated Java class.

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    counter = jfield("int", default=0)
    title = jfield("java.lang.String", default="Hello")
```

You can read and write them like normal Python attributes:

```python
self.counter += 1
self.title = "Updated"
```

Notes:

  * the field is generated on the Java class
  * the value is available through the Python peer object as a normal attribute
  * assigning to the field updates the attached Java instance when possible

### Java-only Field Accessors

If a method is called very often and it only needs to return or update a field value, you can generate a Java-only getter or setter directly from `jfield(...)`.

```python
from extera_utils.classes import jfield, jgetmethod, jsetmethod
 
 
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    shadow_value = jfield(
        "boolean",
        default=False,
        methods=[
            jgetmethod("isShadow"),
            jsetmethod("setShadow"),
        ],
    )
```

This generates:

  * a Java field named `shadow_value`
  * a Java method `isShadow()` which directly returns that field
  * a Java method `setShadow(boolean)` which directly writes that field

These accessor methods do not call into Python at all, so they are useful for very hot paths such as repeated UI queries.

This is especially useful when you want to override a parent getter with a static value:

```python
@java_subclass(SomeFactoryClass)
class OptimizedFactory(Base):
    is_shadow_value = jfield(
        "boolean",
        default=False,
        methods=[jgetmethod("isShadow")],
    )
```

If `SomeFactoryClass` already has `isShadow()`, the generated Java method overrides it in the proxy class and returns the field immediately.

Because generated proxy classes now load through a shared generated-class loader chain, a proxy created later can also reference a proxy class created earlier as a field or method type if you truly need that.

Even so, a stable shared type such as an interface or common base class is still the safer design for public APIs.

## Constructors

Constructor hooks are split into two stages:

  * `@jpreconstructor(...)`: before the parent Java constructor call
  * `@jconstructor(...)`: after the Java object has already been created

### Pre-constructor

Use this when you need to change constructor arguments:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @jpreconstructor(["java.lang.String"])
    def normalize_text(cls, text):
        return [text.strip()]
```

The function should return:

  * `None` to keep original arguments
  * a list or tuple with replacement arguments
  * a single value for a one-argument constructor

### Post-constructor

Use this when the Java object already exists and you want to initialize Python state:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    initialized = jfield("boolean", default=False)
 
    @jconstructor(["java.lang.String"])
    def init_from_text(self, text):
        self.initialized = True
        self.last_text = text
```

### `__init__` and `on_post_init`

The managed DSL also supports generic initialization:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    def __init__(self, text):
        self.python_only_state = text
 
    def on_post_init(self, text):
        print("Java object is ready:", self.java)
```

Initialization order is:

  1. `@jpreconstructor(...)` if matched
  2. Java parent constructor
  3. Python `__init__(...)`
  4. matched `@jconstructor(...)`
  5. `on_post_init(...)`

If you only need a single generic hook, `__init__` is usually enough.

If you need different logic for overloaded constructors, use `@jconstructor(...)`.

### Separate Python init arguments

By default, Python `__init__` receives the same arguments as the Java constructor.

If you need different Python-only initialization data, pass it explicitly through `new_instance(..., init_args=[...])`:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    def __init__(self, title, debug_mode):
        self.title = title
        self.debug_mode = debug_mode
 
 
instance = MySubclass.new_instance(
    "java constructor text",
    init_args=["Python title", True],
)
```

This is useful when the Java constructor signature and the Python peer state do not match exactly.

## Accessing the Attached Java Instance

Inside a managed subclass:

  * `self.java` is the attached Java instance
  * `self.this` is an alias to `self.java`

Example:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride()
    def onAttached(self):
        print(self.java.getClass().getName())
```

If Python cannot find an attribute on the peer object, it falls back to the Java instance:

```python
self.getContext()
self.hashCode()
```

This means most Java calls work directly on the peer object:

```python
instance = MySubclass.new_instance()
instance.hashCode()
instance.getClass().getName()
```

But if a Java API checks the exact runtime type and expects a real Java object parameter, pass `instance.java`.

## Wrapping Existing Java Instances

Sometimes an API hands your generated object back to you as a plain Java object. In that case, use `from_java(...)` to recover the Python peer instead of manually copying state around.

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride()
    def onAttached(self):
        print("Attached!", self.java)
 
 
peer = MySubclass.new_instance()
raw_java = peer.java
 
# Later, possibly in another callback:
same_peer = MySubclass.from_java(raw_java)
same_peer.onAttached()
```

If the peer already exists, `from_java(...)` returns the same Python object. If it does not, the proxy system creates and binds it, unless the object was released with `jrelease()`, in which case it returns `None`.

## Object Lifetime

A managed object and its Java instance stay alive as long as either side uses them. A Java instance which is still registered somewhere, such as a receiver, an observer or a view in a layout, keeps its Python peer and the peer's state, and a peer which your code still refers to keeps its Java instance. Once neither side uses them, both are freed.

When a plugin is unloaded, the objects of the classes it defined are released as described below, so that the plugin's module can be freed even if it forgot to unregister something.

### Releasing an object early

`jrelease()` disconnects a peer from its Java instance. The Java instance stays valid, but its overridden methods then behave like those of the Java base class. Use it when Java may keep the object for longer than you need it:

```python
receiver = MyReceiver.new_instance()
context.registerReceiver(receiver.java, intent_filter)
...
receiver.jrelease()
```

Override `on_release()` to clean up at that point. The Java instance is still connected while it runs:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    def on_release(self):
        self.callback = None
```

### Avoid references back to the Java instance

If a peer stores something which refers back to its own Java instance, such as `self.java` itself, or a Java object holding one of the peer's bound methods, the two keep each other alive until the object is released. When you need the Java instance, get it from `self.java` instead of storing it.

## Passing Python Objects Through Java

The low-level helper `PyObj` is available when you need to carry an arbitrary Python object through Java-facing APIs.

```python
from extera_utils.classes import PyObj
 
 
payload = PyObj.create({"debug": True, "label": "factory args"})
```

`PyObj` is mainly useful when:

  * a Java API needs to carry opaque plugin-owned state
  * you are building low-level adapters or factory layers
  * you want to place Python data into a Java field or something like `UItem.object`

For normal subclassing and method overrides, you usually do not need `PyObj` directly.

## Built-in Java Object Modifiers

The user-friendly behavior is now available directly in the Externagram Chaquopy fork, so you can use it for any Java object or Java class without additional wrappers or imports:

```python
obj.JA.privateField
obj.JIR.method1().method2()
SomeJavaClass.JGS.someStaticProperty
```

### What is available

There are two groups of modifiers.

Persistent modifiers:

  * `JUseGetterAndSetter` / `JGS`
  * `JNotUseGetterAndSetter` / `JNGS`
  * `JAccessAll` / `JA`
  * `JNotAccessAll` / `JNA`

These change the behavior of the current Java object or Java class itself.

Scoped chain modifiers:

  * `JIgnoreResult` / `JIR`
  * `JNotIgnoreResult` / `JNIR`
  * `JSafe` / `JS`
  * `JNotSafe` / `JNS`

These only affect the current access chain and do not permanently mutate the object.

### Default behavior

By default:

  * getter/setter mode is enabled
  * access-all mode is disabled
  * safe mode is disabled
  * ignore-result mode is disabled

That means regular access tries normal Chaquopy behavior first, but field-like names also prefer JavaBean-style accessors when available.

```python
# If getTitle() exists, this uses it by default.
title = obj.title
 
# If setTitle(...) exists, this uses it by default.
obj.title = "Updated"
```

This getter/setter preference applies to:

  * Java object instances
  * Java classes when you work with static getters/setters

### Getter/Setter mode

Use `JNGS` when you want raw field-style access instead of `getX()` / `setX(...)`.

```python
# Disable getter/setter mode only for this object.
obj.JNGS
 
# Now this reads the field path directly instead of trying getResult().
value = obj.result
 
# Turn getter/setter mode back on.
obj.JGS
```

The same works on classes:

```python
# Static getter/setter mode on the class itself.
SomeJavaClass.JGS
caption = SomeJavaClass.caption
SomeJavaClass.caption = "New value"
```

Notes:

  * getter/setter mode is intended for field-like names such as `title`, `result`, `messageObject`
  * names which already look like methods, such as `getClass()` or `isInterface()`, stay method-like and are not remapped again

### AccessAll mode

Use `JA` to allow access to non-public declared members.

```python
# Without JA, private access raises an error.
obj.JA
 
# Private/protected/package-private declared members become accessible.
secret = obj.secretField
obj.secretMethod()
 
# Turn the mode off again on the same object.
obj.JNA
```

This also works on Java classes for static members:

```python
SomeJavaClass.JA
 
# Access a private static field or method.
token = SomeJavaClass.secretToken
SomeJavaClass.rebuildCache()
 
SomeJavaClass.JNA
```

Internally, the Chaquopy fork caches declared methods, fields, and nested classes per Java class, so `JA` does not re-scan reflection metadata every time.

### IgnoreResult mode

Use `JIR` when you want a method chain to keep returning the original receiver instead of the Java return value.

```python
# All three calls are made on obj.
obj.JIR.setTitle("Hello").requestLayout().invalidate()
 
# Normal behavior resumes outside that chain.
result = obj.methodThatReturnsSomething()
```

You can turn it off in the same chain with `JNIR`:

```python
# The first three calls return the original object.
# method4() then returns its real Java result.
value = obj.JIR.method1().method2().method3().JNIR.method4()
```

### Safe mode

Use `JS` when you want safe optional-style chaining.

If some attribute or method is missing anywhere in the chain, the result becomes `JNone` instead of raising immediately.

```python
value = obj.JS.someAttr.someOtherAttr.someMethod()
```

Safe mode also propagates to Java objects returned during the chain, so you do **not** need to write `.JS` again after every step.

It also treats `None` as safe-empty and converts it to `JNone` while safe mode is active.

```python
# If getInfo() returns None, the chain still stays safe.
value = obj.JS.info.details.title
 
# If any step is missing, you get JNone instead of an exception.
value = obj.JS.missing.anything.call()
```

You can disable it mid-chain with `JNS`.

### `JNone`

`JNone` is the sentinel returned by safe chains when access fails or a safe-chain step returns `None`.

Properties:

  * falsy in boolean context
  * keeps swallowing attribute access
  * keeps swallowing calls
  * can be compared by identity when needed

```python
from java import JNone
 
result = obj.JS.maybeNull.value
 
if result is JNone:
    print("No value")
```

### Object-level and class-level examples

```python
# Instance-level flags persist on this object.
obj.JNGS
obj.JA
 
# Direct field access, including non-public declared fields.
raw = obj.result
secret = obj.secretField
 
# Restore the defaults for this object.
obj.JNA
obj.JGS
```

```python
# Class-level flags persist on this Java class.
SomeJavaClass.JGS
SomeJavaClass.JA
 
# Static getter/setter or static field access.
current = SomeJavaClass.currentItem
SomeJavaClass.currentItem = item
hidden = SomeJavaClass.hiddenCache
 
SomeJavaClass.JNA
```

```python
# Chain-scoped modifiers do not persist.
res = obj.JIR.prepare().bind().show().JNIR.getResult()
 
# Safe chaining over Java objects and None.
title = obj.JS.info.card.title
```

### Practical behavior summary

  * `JGS` / `JNGS` and `JA` / `JNA` are persistent for the current object or Java class
  * `JIR` / `JNIR` and `JS` / `JNS` only affect the current chain
  * `dir(obj)` and `dir(SomeJavaClass)` expose the modifier names
  * `JA` works with fields, methods, and nested classes/interfaces declared on the Java class
  * `JSafe` propagates through returned Java objects automatically
  * `JSafe` converts `None` to `JNone` inside the safe chain

### Limitations

This is still convenience access, not a full replacement for manual signature-driven reflection.

Keep in mind:

  * if exact overload control matters, explicit Java reflection is still safer
  * non-public lookup is based on declared members and class metadata from the Chaquopy fork
  * getter/setter preference is only for field-like names, not for method-like names such as `getX`

If you need precise overload selection, use regular reflection:

```python
clazz = obj.getClass()
method = clazz.getDeclaredMethod("setValue", jint)
method.setAccessible(True)
method.invoke(obj, 5)
 
# or
obj.JA  # enable Access All mode
obj.value = 5  # use setValue(5)
```

## Full Example

```python
from extera_utils.classes import (
    Base,
    java_subclass,
    joverride,
    joverload,
    jmethod,
    jfield,
    jconstructor,
    jpreconstructor,
)
 
 
@java_subclass(SomeJavaClass)
class DemoProxy(Base):
    counter = jfield("int", default=0)
    title = jfield("java.lang.String", default="Demo")
 
    @jpreconstructor(["java.lang.String", "int"])
    def normalize_args(cls, title, counter):
        return [title.strip(), max(0, counter)]
 
    def __init__(self, title, counter):
        self.python_state = {"created_with": title}
 
    @jconstructor(["java.lang.String", "int"])
    def init_fields(self, title, counter):
        self.title = title
        self.counter = counter
 
    @joverride()
    def onAttached(self):
        self.counter += 1
        return super().onAttached()
 
    @joverload("setValue", ["int"])
    def set_value_int(self, value):
        self.counter = value
        return super().set_value_int(value)
 
    @joverload("setValue", ["java.lang.String"])
    def set_value_text(self, value):
        self.title = value
 
    @jmethod("debugLabel", "java.lang.String", ["int"])
    def build_debug_label(self, extra):
        return f"{self.title} ({self.counter}, extra={extra})"
 
 
instance = DemoProxy.new_instance("  Example  ", 5)
java_instance = instance.java
```

## Full Smoke Test Example

This example is designed to verify almost the entire class proxy API in one place:

  * subclass generation
  * Java field creation
  * constructor preprocessing
  * post-constructor hooks
  * overloaded method overrides
  * `super()` calls
  * custom Java methods
  * access to the attached Java instance

It uses `java.util.ArrayList`, because it is a normal subclassable Java class with overloaded methods and constructors.

```python
from base_plugin import BasePlugin
from extera_utils.classes import (
    Base,
    java_subclass,
    jfield,
    joverride,
    joverload,
    jmethod,
    jconstructor,
    jpreconstructor,
)
from java.util import ArrayList
 
 
@java_subclass(ArrayList)
class DebugList(Base):
    add_calls = jfield("int", default=0)
    remove_calls = jfield("int", default=0)
    last_action = jfield("java.lang.String", default="created")
 
    @jpreconstructor(["int"])
    def normalize_capacity(cls, capacity):
        # Ensure capacity is never negative before Java constructor runs.
        return [max(0, capacity)]
 
    def __init__(self, *args):
        # Python-only state. This is not a Java field.
        self.history = []
 
    @jconstructor(["int"])
    def init_with_capacity(self, capacity):
        self.last_action = f"capacity:{capacity}"
        self.history.append(f"ctor(capacity={capacity})")
 
    @joverride()
    def size(self):
        # Example of overriding a normal method and still delegating to parent.
        current = super().size()
        self.history.append(f"size() -> {current}")
        return current
 
    @joverload("add", ["java.lang.Object"])
    def add_item(self, value):
        self.add_calls += 1
        self.last_action = f"add:{value}"
        self.history.append(f"add(value={value})")
        return super().add_item(value)
 
    @joverload("add", ["int", "java.lang.Object"])
    def add_at_index(self, index, value):
        self.add_calls += 1
        self.last_action = f"add_at:{index}:{value}"
        self.history.append(f"add(index={index}, value={value})")
        return super().add_at_index(index, value)
 
    @joverload("remove", ["int"])
    def remove_at_index(self, index):
        self.remove_calls += 1
        self.last_action = f"remove_at:{index}"
        self.history.append(f"remove(index={index})")
        return super().remove_at_index(index)
 
    @joverload("remove", ["java.lang.Object"])
    def remove_item(self, value):
        self.remove_calls += 1
        self.last_action = f"remove:{value}"
        self.history.append(f"remove(value={value})")
        return super().remove_item(value)
 
    @jmethod("debugSummary", "java.lang.String", [])
    def debug_summary(self):
        return (
            f"size={self.size()}, "
            f"add_calls={self.add_calls}, "
            f"remove_calls={self.remove_calls}, "
            f"last_action={self.last_action}"
        )
 
    @jmethod("historyDump", "java.lang.String", [])
    def history_dump(self):
        return " | ".join(self.history)
 
 
class ProxyDebugPlugin(BasePlugin):
    def on_plugin_load(self):
        debug_list = DebugList.new_instance(4)
 
        debug_list.add("one")
        debug_list.add("two")
        debug_list.add(1, "inserted")
        debug_list.remove("two")
 
        self.logger.info(f"Class name: {debug_list.getClass().getName()}")
        self.logger.info(f"Java size(): {debug_list.size()}")
        self.logger.info(f"Python field add_calls: {debug_list.add_calls}")
        self.logger.info(f"Python field remove_calls: {debug_list.remove_calls}")
        self.logger.info(f"Java method debugSummary(): {debug_list.debugSummary()}")
        self.logger.info(f"Java method historyDump(): {debug_list.historyDump()}")
 
        # Access raw Java instance from the Python peer if needed.
        self.logger.info(f"Same object hash: {debug_list.java.hashCode()}")
```

### What to expect

If everything works, this example should confirm that:

  * `DebugList.new_instance(4)` selects the `ArrayList(int)` constructor
  * `DebugList.new_instance(4)` returns the Python peer object, and `debug_list.java` is the raw Java instance
  * `@jpreconstructor(["int"])` runs before Java construction
  * `@jconstructor(["int"])` runs after Java construction
  * `add_calls`, `remove_calls`, and `last_action` behave like Java-backed fields
  * the correct overloaded `add` and `remove` handlers are called
  * `super()` reaches the original `ArrayList` implementation
  * `debugSummary()` and `historyDump()` exist as real Java methods on the generated class

## Custom Settings Example

The settings SDK uses this system internally for custom setting factories.

Here is a simplified pattern:

```python
from client_utils import get_media_data_controller
from base_plugin import BasePlugin
from ui.settings import Text, Custom
from extera_utils.classes import Base, java_subclass, joverride
 
from android.util import TypedValue
from android.view import View, Gravity
from android.widget import LinearLayout, FrameLayout, TextView
from android.content import Context
 
from org.telegram.messenger import AndroidUtilities
from org.telegram.ui.ActionBar import Theme
from org.telegram.ui.Components import LayoutHelper, BackupImageView, UItem
 
from com.exteragram.messenger.plugins import PluginsController, Plugin
from com.exteragram.messenger.plugins.models import CustomSetting
 
MATCH_PARENT = -1
WRAP_CONTENT = -2
 
@java_subclass(FrameLayout)
class PluginCell(Base):
    def on_post_init(self, context: Context):
        self.need_divider = False
        self.setWillNotDraw(False)
 
        self.icon_view = BackupImageView(context)
        self.icon_view.setRoundRadius(AndroidUtilities.dp(8))
        self.addView(self.icon_view, LayoutHelper.createFrame(48, 48, Gravity.LEFT | Gravity.CENTER_VERTICAL, 16, 0, 0, 0))
 
        text_layout = LinearLayout(context)
        text_layout.setOrientation(LinearLayout.VERTICAL)
 
        self.name_text_view = TextView(context)
        self.name_text_view.setTextColor(Theme.getColor(Theme.key_windowBackgroundWhiteBlackText))
        self.name_text_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
        self.name_text_view.setLines(1)
        self.name_text_view.setSingleLine(True)
 
        self.author_text_view = TextView(context)
        self.author_text_view.setTextColor(Theme.getColor(Theme.key_windowBackgroundWhiteGrayText))
        self.author_text_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 13)
        self.author_text_view.setLines(1)
        self.author_text_view.setSingleLine(True)
 
        text_layout.addView(self.name_text_view, LayoutHelper.createLinear(MATCH_PARENT, WRAP_CONTENT))
        text_layout.addView(self.author_text_view, LayoutHelper.createLinear(MATCH_PARENT, WRAP_CONTENT, 0, 2, 0, 0))
        self.addView(text_layout, LayoutHelper.createFrame(MATCH_PARENT, WRAP_CONTENT, Gravity.LEFT | Gravity.CENTER_VERTICAL, 76, 0, 16, 0))
 
    def set_data(self, plugin: Plugin, divider: bool):
        self.need_divider = divider
        self.name_text_view.setText(plugin.getName() or "Unknown Plugin")
 
        version = plugin.getVersion() or "1.0"
        author = plugin.getAuthor() or "Unknown"
        self.author_text_view.setText(f"v{version} • {author}")
 
        show_icon = plugin.getPack() is not None and plugin.getIndex() >= 0
        pack, index = (plugin.getPack(), plugin.getIndex()) if show_icon else ("exteraPlugins", 0)
        get_media_data_controller().setPlaceholderImageByIndex(self.icon_view, pack, index, "100_100")
 
        self.requestLayout()
    
    @joverride()
    def onMeasure(self, widthMeasureSpec: int, heightMeasureSpec: int):
        super().onMeasure(
            View.MeasureSpec.makeMeasureSpec(View.MeasureSpec.getSize(widthMeasureSpec), View.MeasureSpec.EXACTLY),
            View.MeasureSpec.makeMeasureSpec(AndroidUtilities.dp(68), View.MeasureSpec.EXACTLY),
        )
 
 
@java_subclass(CustomSetting.Factory)
class PluginCellSettingFactory(Base):
    @joverride()
    def create(self, plugin: Plugin, setting: CustomSetting, args):
        if not args:
            return None
        
        item = UItem.ofFactory(self.java_class())
        item.object = args
        return item
 
    @joverride()
    def createView(self, context, list_view, current_account, class_guid, resources_provider):
        view = PluginCell.new_instance(context)
        return view.java
 
    @joverride()
    def bindView(self, view, item, divider, adapter, list_view):
        if not item.object:
            return False
 
        PluginCell.from_java(view).set_data(item.object, divider)
 
 
class TestPlugin(BasePlugin):
    def on_plugin_load(self):
        # plugin_cell_setting_factory_instance
        self.pcsfi = PluginCellSettingFactory.new_java_instance()
        UItem.UItemFactory.setup(self.pcsfi)
 
    def create_settings(self):
        plugins = PluginsController.getInstance().plugins.values().toArray()
        if not plugins:
            return [
                Text("Restart plugin please, no plugins found")
            ]
 
        return [
            Custom(
                factory=self.pcsfi,
                factory_args=_p
            ) for _p in plugins
        ]
```

If you instantiate this proxy with `FactoryProxy.new_instance()`, the result is the Python peer. Pass `factory.java` into APIs which require `CustomSetting.Factory`.

For a higher-level settings guide, see [Plugin Settings](</docs/plugin-settings>).

## Advanced Notes

### Method Resolution Order

For managed overrides, the proxy system installs a Python bridge so that `super()` calls the parent Java implementation.

Because of this, it is best to use `super()` only for methods marked with `@joverride(...)` or `@joverload(...)`.

### Exact Signature Dispatch

When Java calls Python, the proxy system uses a full signature key internally:

```python
methodName:int:java.lang.String
```

This is what makes overloaded methods and constructors work correctly.

### Primitive Types

Supported primitive type names:

  * `void`
  * `boolean`
  * `byte`
  * `char`
  * `short`
  * `int`
  * `long`
  * `float`
  * `double`

### When To Use It

Use class proxy when:

  * a Java API requires subclassing instead of interface proxying
  * you need parent-method calls with `super()`
  * you need overload-specific behavior
  * you want Java fields or extra Java methods on the generated object

Do not use it when a simple `dynamic_proxy(...)` is enough for an interface.

Also make sure the base Java class is actually subclassable. Final classes cannot be used as a proxy base class.

## Troubleshooting

### `super()` does not work

Check that:

  * the method is declared with `@joverride(...)` or `@joverload(...)`
  * the method name or signature matches the Java parent method

### My overload is not called

Check that:

  * the Java name is correct
  * the argument types match the real Java signature exactly
  * primitive types use primitive names such as `"int"` instead of wrapper classes

### Constructor hook is not called

Check that:

  * `@jpreconstructor(...)` or `@jconstructor(...)` uses the correct constructor signature
  * for wildcard constructor hooks, your annotations are either complete or you intentionally omitted the signature

### Field value is always `None`

Check that:

  * the field was declared with `jfield(...)`
  * the field type is valid
  * the default value is compatible with the declared Java type

### Class generation fails immediately

Check that:

  * the base class is not `final`
  * constructor signatures match a real constructor
  * overload signatures match real Java method argument types

[PreviousLogging](</docs/logging>)[NextXposed Method Hooking](</docs/xposed-hooking>)


---


# === DOCUMENT: Xposed Method Hooking (/docs/xposed-hooking) ===

---
title: "Xposed Method Hooking"
source: "https://plugins.exteragram.app/docs/xposed-hooking"
---

# Xposed Method Hooking

Hook Java methods and constructors from Python plugins using MethodHook, MethodReplacement, BaseHook, and filters.

## Introduction

Xposed-style method hooking lets your plugin intercept method calls inside the app, modify arguments, inspect or replace return values, or skip the original implementation entirely.

This is one of the most powerful parts of the plugin SDK, and also one of the easiest to misuse, so it is worth being explicit about the available styles.

## Hook Handler Styles

You can provide hook logic in three ways.

### 1\. `MethodHook`

Use `MethodHook` when you want `before_hooked_method(...)` and/or `after_hooked_method(...)`.

```python
from base_plugin import MethodHook
 
 
class TitleLoggerHook(MethodHook):
    def before_hooked_method(self, param):
        title = param.args[0]
        print(f"ActionBar title is about to become: {title}")
 
    def after_hooked_method(self, param):
        print(f"Method finished with result: {param.getResult()}")
```

### 2\. `MethodReplacement`

Use `MethodReplacement` when you want to fully replace the original Java method.

```python
from base_plugin import MethodReplacement
 
 
class TitleReplacer(MethodReplacement):
    def replace_hooked_method(self, param):
        print("Original method was replaced.")
        return None
```

### 3\. Functional hooks or `BaseHook`

For simpler cases, you can skip creating a full class and pass `before=` and/or `after=` directly:

```python
self.hook_method(
    some_method,
    before=lambda param: self.logger.info("Method is about to run!"),
    after=lambda param: self.logger.info(f"Method finished with result: {param.getResult()}"),
)
```

Internally the SDK wraps that style into `BaseHook`, which is a small helper subclass of `MethodHook`.

## The `param` Object

All hook callbacks receive `de.robv.android.xposed.XC_MethodHook.MethodHookParam`.

The most useful fields and methods are:

  * `param.thisObject`: instance on which the method was called, or `None` for static methods
  * `param.args`: method argument array; you can read and modify it
  * `param.method`: the hooked `Method` or `Constructor`
  * `param.getResult()`: return value of the original method, available after execution
  * `param.setResult(value)`: sets a result manually

If you call `param.setResult(...)` inside `before_hooked_method(...)`, the original method is skipped.

```python
def before_hooked_method(self, param):
    if param.args[0] == "blocked":
        param.setResult(None)
```

## Filters

Filters decide whether your hook callback should run at all.

You can use them in two styles:

  * `@hook_filters(...)` on class-based hook methods
  * `before_filters=[...]` and `after_filters=[...]` for functional hooks

```python
from base_plugin import HookFilter
 
 
self.hook_method(
    some_method,
    before=lambda p: self.logger.info("Arg 0 is null!"),
    before_filters=[HookFilter.ArgumentIsNull(0)],
)
```

### Available `HookFilter` helpers

  * `HookFilter.RESULT_IS_NULL`
  * `HookFilter.RESULT_IS_TRUE`
  * `HookFilter.RESULT_IS_FALSE`
  * `HookFilter.RESULT_NOT_NULL`
  * `HookFilter.ResultIsInstanceOf(clazz)`
  * `HookFilter.ResultEqual(value)`
  * `HookFilter.ResultNotEqual(value)`
  * `HookFilter.ArgumentIsNull(index)`
  * `HookFilter.ArgumentNotNull(index)`
  * `HookFilter.ArgumentIsFalse(index)`
  * `HookFilter.ArgumentIsTrue(index)`
  * `HookFilter.ArgumentIsInstanceOf(index, clazz)`
  * `HookFilter.ArgumentEqual(index, value)`
  * `HookFilter.ArgumentNotEqual(index, value)`
  * `HookFilter.Condition(condition, object=None)`
  * `HookFilter.Or(*filters)`

### Class-based filter example

```python
from base_plugin import HookFilter, MethodHook, hook_filters
 
 
class ExampleHook(MethodHook):
    # Run only if the first argument is null.
    @hook_filters(HookFilter.ArgumentIsNull(0))
    def before_hooked_method(self, param):
        ...
 
    # Run only if the original result is not null.
    @hook_filters(HookFilter.RESULT_NOT_NULL)
    def after_hooked_method(self, param):
        ...
```

### MVEL condition filter example

```python
from base_plugin import HookFilter, MethodHook, hook_filters
 
 
class ConditionalHook(MethodHook):
    @hook_filters(
        HookFilter.Condition(
            "param.args[0] == object || this instanceof android.view.View",
            object=500,
        )
    )
    def before_hooked_method(self, param):
        ...
```

## The Hooking Process

### 1\. Find the target method or constructor

You usually start with `find_class(...)`, then call Java reflection directly on the returned `Class`.

```python
from hook_utils import find_class
 
 
ActionBarClass = find_class("org.telegram.ui.ActionBar.ActionBar")
if not ActionBarClass:
    self.logger.info("ActionBar class not found!")
    return
 
 
# Example method: public void setTitle(CharSequence title)
CharSequenceClass = find_class("java.lang.CharSequence")
set_title_method = ActionBarClass.getDeclaredMethod("setTitle", CharSequenceClass)
set_title_method.setAccessible(True)
 
 
# Example constructor: public ActionBar(Context context)
ContextClass = find_class("android.content.Context")
action_bar_constructor = ActionBarClass.getDeclaredConstructor(ContextClass)
action_bar_constructor.setAccessible(True)
```

> > [!NOTE] **Do not call `getClass()` on the `Class` object** > find_class(...)already returns a JavaClass. CallgetDeclaredMethod(...)orgetDeclaredConstructor(...)on that object directly.

### 2\. Implement the hook handler

```python
from base_plugin import MethodHook, MethodReplacement
 
 
class TitleLoggerHook(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin
 
    def before_hooked_method(self, param):
        title = param.args[0]
        self.plugin.log(f"ActionBar title is being set to: {title}")
        param.args[0] = f"[Hooked] {title}"
 
    def after_hooked_method(self, param):
        self.plugin.log("ActionBar title has been set.")
 
 
class TitleReplacer(MethodReplacement):
    def __init__(self, plugin):
        self.plugin = plugin
 
    def replace_hooked_method(self, param):
        self.plugin.log("ActionBar.setTitle() was replaced.")
        return None
```

### 3\. Apply the hook

```python
ActionBarClass = find_class("org.telegram.ui.ActionBar.ActionBar")
CharSequenceClass = find_class("java.lang.CharSequence")
 
set_title_method = ActionBarClass.getDeclaredMethod("setTitle", CharSequenceClass)
set_title_method.setAccessible(True)
 
handler_instance = TitleLoggerHook(self)
self.unhook_obj = self.hook_method(set_title_method, handler_instance, priority=10)
 
if self.unhook_obj:
    self.logger.info("Successfully hooked ActionBar.setTitle()")
```

Hooks are removed automatically when your plugin unloads, but you can also unhook manually.

### 4\. Hook all methods or all constructors

```python
MyViewClass = find_class("com.example.MyCustomView")
unhook_list = self.hook_all_methods(MyViewClass, "onMeasure", TitleLoggerHook(self))
 
if unhook_list:
    self.logger.info(f"Successfully hooked {len(unhook_list)} onMeasure overload(s).")
```

Or:

```python
SomeClass = find_class("com.example.SomeClass")
constructor_unhooks = self.hook_all_constructors(SomeClass, TitleLoggerHook(self))
```

### 5\. Manual unhooking

```python
if self.unhook_obj:
    self.unhook_method(self.unhook_obj)
    self.unhook_obj = None
```

If you used `hook_all_methods(...)` or `hook_all_constructors(...)`, iterate over the returned list and unhook each item.

## Practical Examples

### Example 1: modify arguments before the call

```python
from base_plugin import MethodHook
from hook_utils import find_class
from java import jint
 
 
class ToastHook(MethodHook):
    def before_hooked_method(self, param):
        # Method signature: makeText(Context context, CharSequence text, int duration)
        original_text = param.args[1]
        param.args[1] = f"(Plugin) {original_text}"
 
 
ToastClass = find_class("android.widget.Toast")
ContextClass = find_class("android.content.Context")
CharSequenceClass = find_class("java.lang.CharSequence")
 
make_text_method = ToastClass.getDeclaredMethod("makeText", ContextClass, CharSequenceClass, jint)
self.hook_method(make_text_method, ToastHook())
```

### Example 2: change the return value after the call

```python
from base_plugin import MethodHook
from hook_utils import find_class
 
 
class BuildVarsHook(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin
 
    def after_hooked_method(self, param):
        original_result = param.getResult()
        self.plugin.log(f"Original result: {original_result}")
        param.setResult(False)
 
 
BuildVarsClass = find_class("org.telegram.messenger.BuildVars")
is_main_app_method = BuildVarsClass.getDeclaredMethod("isMainApp")
self.hook_method(is_main_app_method, BuildVarsHook(self))
```

### Example 3: skip the original call in `before_hooked_method`

```python
from base_plugin import MethodHook
from hook_utils import find_class
from java.lang import Boolean, Long
 
 
class FormatFileSizeHook(MethodHook):
    def before_hooked_method(self, param):
        size = param.args[0]
 
        if size < 1024:
            # Skip the original method completely.
            param.setResult(f"{size} bytes (edited)")
 
 
AndroidUtilitiesClass = find_class("org.telegram.messenger.AndroidUtilities")
format_file_size_method = AndroidUtilitiesClass.getDeclaredMethod(
    "formatFileSize",
    Long.TYPE,
    Boolean.TYPE,
    Boolean.TYPE,
)
self.hook_method(format_file_size_method, FormatFileSizeHook())
```

### Example 4: fully replace a method

```python
from java.lang import String as JString
 
from base_plugin import MethodReplacement
from hook_utils import find_class
 
 
class NoOpLogger(MethodReplacement):
    def replace_hooked_method(self, param):
        # Original method is never called.
        return None
 
 
FileLogClass = find_class("org.telegram.messenger.FileLog")
log_method = FileLogClass.getDeclaredMethod("d", JString)
self.hook_method(log_method, NoOpLogger())
```

> > [!NOTE] **Return values in `MethodReplacement`** > Yourreplace_hooked_method(...)becomes the whole implementation. Return a value compatible with the original Java method:NoneforvoidPythonbool/int/floatfor primitive Java returnscompatible Java or Python object for reference returns

[PreviousClass Proxy](</docs/class-proxy>)[NextAndroid Utilities](</docs/android-utils>)


---


# === DOCUMENT: Hook Utilities (Reflection) (/docs/hook-utils) ===

---
title: "Hook Utilities (Reflection)"
source: "https://plugins.exteragram.app/docs/hook-utils"
---

# Hook Utilities (Reflection)

A set of utility functions for performing Java reflection, allowing you to find classes and access or modify private fields and methods.

The `hook_utils.py` module provides essential tools for interacting with the underlying Java code of the application via reflection. This is particularly useful for advanced Xposed hooking when you need to access non-public members of a class.

> > [!NOTE] **Use with Caution** > Reflection is a powerful but fragile technique. It can break if the underlying application code changes. Always include error handling (e.g.,try-exceptblocks) when using these functions and check forNonereturn values.

## `find_class(class_name: str)`

Safely finds and returns a Java class object by its fully qualified name.

  * **`class_name`** : The full name of the class, including the package (e.g., `"org.telegram.ui.ActionBar.ActionBar"`).
  * **Returns** : A Java `Class` object if found, otherwise `None`.

### Example

```python
from hook_utils import find_class
 
# Find the ActionBar class
ActionBarClass = find_class("org.telegram.ui.ActionBar.ActionBar")
 
if ActionBarClass:
    self.logger.info(f"Successfully found class: {ActionBarClass.getName()}")
else:
    self.logger.info("Could not find ActionBar class.")
```

## `get_private_field(obj: JavaObject, field_name: str)`

Accesses and retrieves the value of a private (or public) instance field from a given object. It searches the entire class hierarchy.

  * **`obj`** : The Java object instance from which to get the field.
  * **`field_name`** : The name of the field to access.
  * **Returns** : The value of the field if found, otherwise `None`.

### Example

Assuming `chatActivity` is an instance of `org.telegram.ui.ChatActivity`.

```python
from hook_utils import get_private_field
 
# Get the value of the private 'chatListView' field from a ChatActivity instance
chat_list_view = get_private_field(chatActivity, "chatListView")
 
if chat_list_view:
    self.logger.info("Successfully accessed chatListView.")
```

## `set_private_field(obj: JavaObject, field_name: str, new_value: Any)`

Modifies the value of a private (or public) instance field on a given object.

  * **`obj`** : The Java object instance to modify.
  * **`field_name`** : The name of the field to modify.
  * **`new_value`** : The new value to assign to the field.
  * **Returns** : `True` if the field was set successfully, `False` otherwise.

### Example

```python
from hook_utils import set_private_field
 
# Change the value of a 'verified' field on a user object
user_object = ...
success = set_private_field(user_object, "verified", True)
 
if success:
    self.logger.info("User is now verified!")
```

## `get_static_private_field(clazz: JavaClass, field_name: str)`

Accesses and retrieves the value of a `static` private (or public) field from a given class.

  * **`clazz`** : The class, as returned by `find_class`. A `java.lang.Class` object works too.
  * **`field_name`** : The name of the static field.
  * **Returns** : The value of the field if found, otherwise `None`.

### Example

```python
from hook_utils import find_class, get_static_private_field
 
# Get the static 'configLoaded' field from ExteraConfig
ExteraConfigClass = find_class("com.exteragram.messenger.ExteraConfig")
if ExteraConfigClass:
    config_loaded = get_static_private_field(ExteraConfigClass, "configLoaded")
    self.logger.info(f"Config loaded: {config_loaded}")
```

## `set_static_private_field(clazz: JavaClass, field_name: str, new_value: Any)`

Modifies the value of a `static` private (or public) field on a given class.

  * **`clazz`** : The class, as returned by `find_class`. A `java.lang.Class` object works too.
  * **`field_name`** : The name of the static field to modify.
  * **`new_value`** : The new value to assign.
  * **Returns** : `True` if successful, `False` otherwise.

### Example

```python
from hook_utils import find_class, set_static_private_field
 
# Modify a static configuration flag
BuildVarsClass = find_class("org.telegram.messenger.BuildVars")
if BuildVarsClass:
    success = set_static_private_field(BuildVarsClass, "DEBUG_VERSION", True)
    if success:
        self.logger.info("DEBUG_VERSION has been enabled.")
```

[PreviousText Formatting](</docs/text-formatting>)[NextFile Utilities](</docs/file-utils>)


---


# === DOCUMENT: Android Utilities (/docs/android-utils) ===

---
title: "Android Utilities"
source: "https://plugins.exteragram.app/docs/android-utils"
---

# Android Utilities

Helpers for UI-thread work, Android listeners, clipboard access, and Java Runnable wrappers.

This module provides small but very frequently used helpers for Android-side work inside plugins.

Most of the time you will use it for:

  * running code on the UI thread
  * attaching simple click listeners
  * copying text to the clipboard

For logging, use [`self.logger`](</docs/logging>).

## Wrappers for Java Interfaces

These wrappers let you pass Python callables into Android APIs that expect Java listener objects.

### R (Runnable Proxy)

`R(fn)` creates a real Java `Runnable` backed by your Python callable.

This is useful when some Android or Telegram API expects a `java.lang.Runnable` instance, for example `view.post(...)`, delayed callbacks, or various internal helper methods.

```python
from android_utils import R, log
 
 
def my_task():
    log("Runnable task executed.")
 
 
# Create a Java Runnable instance backed by Python.
runnable_instance = R(my_task)
 
# Example usage with an API that expects Runnable.
some_view = ...
some_view.post(runnable_instance)
```

> > [!NOTE] **`run_on_ui_thread` already wraps your callable** > run_on_ui_thread(...)accepts a normal Python callable and internally converts it to a JavaRunnablefor you. UseR(...)when some other API explicitly requires aRunnableobject.

### OnClickListener

`OnClickListener(fn)` is a convenience wrapper for `android.view.View.OnClickListener`.

```python
from android_utils import OnClickListener, log
from android.view import View
 
 
def handle_button_click(view: View):
    log(f"Button {view.getId()} was clicked.")
 
 
button = ...
button.setOnClickListener(OnClickListener(handle_button_click))
```

Your callable receives the clicked `View` as its only argument.

### OnLongClickListener

`OnLongClickListener(fn)` is a convenience wrapper for `android.view.View.OnLongClickListener`.

```python
from android_utils import OnLongClickListener, log
from android.view import View
 
 
def handle_button_long_click(view: View):
    log(f"Button {view.getId()} was long-clicked.")
    return True
 
 
button = ...
button.setOnLongClickListener(OnLongClickListener(handle_button_long_click))
 
# Or with a lambda:
button.setOnLongClickListener(OnLongClickListener(lambda v: (log("Long click!"), True)[1]))
```

The wrapped Python callable is expected to return a boolean:

  * `True`: consume the long-click event
  * `False`: let Android continue processing it

## Utility Functions

### run_on_ui_thread

`run_on_ui_thread(func, delay=0)` schedules a Python callable on the Android UI thread.

Any UI mutation should happen there: changing text, updating views, showing dialogs, attaching adapters, and so on.

```python
from android_utils import run_on_ui_thread
 
 
def update_ui_content():
    text_view = ...
    text_view.setText("Updated from Python on UI thread")
 
 
# Run immediately (or as soon as possible) on the UI thread.
run_on_ui_thread(update_ui_content)
 
# Run with a delay of 500 milliseconds.
run_on_ui_thread(update_ui_content, 500)
```

Parameters:

  * `func`: Python callable to execute
  * `delay`: optional delay in milliseconds

### log (legacy)

> > [!NOTE] **Legacy** > log()is kept for existing plugins. New code should useself.logger: it has levels, a per-plugin logcat tag, tracebacks and object dumps.

`log(data)` writes to logcat at debug level:

  * simple values such as `str`, `int`, `float`, `bool`, and `None` are logged as text
  * any other object is logged as its [`dump()`](</docs/logging#inspecting-objects>)

```python
from android_utils import log
 
 
log("This is a simple log message.")
log(user_object)
```

### copy_to_clipboard

`copy_to_clipboard(text)` copies a string to the Android clipboard and, if the operation succeeds, shows the standard "copied" bulletin.

```python
from android_utils import copy_to_clipboard
 
 
copy_to_clipboard("https://t.me/exteraPlugins")
```

This is handy for deeplinks, debug info, exported IDs, or user-facing share actions.

[PreviousXposed Method Hooking](</docs/xposed-hooking>)[NextClient Utilities](</docs/client-utils>)


---


# === DOCUMENT: Client Utilities (/docs/client-utils) ===

---
title: "Client Utilities"
source: "https://plugins.exteragram.app/docs/client-utils"
---

# Client Utilities

Helpers for queues, requests, notifications, controllers, fragments, and sending or editing messages.

This module contains the higher-level helpers most plugins use when they need to interact with Telegram internals without dropping into raw Java code every time.

Typical uses:

  * queueing work off the UI thread
  * sending Telegram requests
  * sending or editing messages and media
  * reaching the current fragment or account controllers
  * listening to `NotificationCenter`

## Queues (Background Threads)

Use `run_on_queue(...)` for work that should not block the UI thread: network requests, file I/O, heavy parsing, or long computations.

```python
import time
 
from android_utils import log
from client_utils import run_on_queue
 
 
def my_long_task(parameter: str):
    log(f"Task started with: {parameter}")
    time.sleep(5)  # Simulate a long operation.
    log(f"Task finished for: {parameter}")
 
 
# Run on the default PLUGINS_QUEUE.
run_on_queue(lambda: my_long_task("some_data"))
```

You can target a specific queue and add a delay in milliseconds:

```python
from client_utils import GLOBAL_QUEUE, run_on_queue
 
 
# Run on GLOBAL_QUEUE after a 2.5 second delay.
run_on_queue(lambda: my_long_task("other_data"), GLOBAL_QUEUE, 2500)
```

Available queue-name constants:

```python
STAGE_QUEUE = "stageQueue"
GLOBAL_QUEUE = "globalQueue"
CACHE_CLEAR_QUEUE = "cacheClearQueue"
SEARCH_QUEUE = "searchQueue"
PHONE_BOOK_QUEUE = "phoneBookQueue"
THEME_QUEUE = "themeQueue"
EXTERNAL_NETWORK_QUEUE = "externalNetworkQueue"
PLUGINS_QUEUE = "pluginsQueue"
```

If you need the raw Java `DispatchQueue`, use `get_queue_by_name(...)`:

```python
from client_utils import PLUGINS_QUEUE, get_queue_by_name
 
 
plugins_dispatch_queue = get_queue_by_name(PLUGINS_QUEUE)
if plugins_dispatch_queue:
    # You can call DispatchQueue methods directly here.
    pass
```

## Sending Telegram API Requests

`send_request(request, fn)` sends a raw Telegram `TLObject` through the `ConnectionsManager` of the account selected in the UI, or of `account=` when you pass one. Your callback runs with that account in scope, so helpers you call from it can follow it — see [Multi-account](</docs/multi-account>).

You pass a normal Python callback. The SDK wraps it into a `RequestDelegate` for you.

```python
from java.lang import Integer
from org.telegram.tgnet import TLRPC
 
from android_utils import log
from client_utils import send_request
 
 
def handle_read_contents_response(response, error):
    if error:
        log(f"Error reading message contents: {error.text}")
        return
 
    if response and isinstance(response, TLRPC.TL_messages_affectedMessages):
        log(f"Successfully read contents. PTS: {response.pts}, Count: {response.pts_count}")
    else:
        log(f"Unexpected response type: {type(response)}")
 
 
# Create the request object.
req = TLRPC.TL_messages_readMessageContents()
req.id.add(Integer(12345))
 
# Send the request. The return value is the ConnectionsManager request id.
request_id = send_request(req, handle_read_contents_response)
log(f"Sent TL_messages_readMessageContents, request ID: {request_id}")
```

You usually do not need to instantiate `RequestCallback` manually unless you specifically want the proxy class itself.

## Sending Messages and Media

The `send_*` helpers build the proper `SendMessageParams`, prepare media objects when needed, and send the final operation on the correct thread.

All `send_*` helpers accept optional `parse_mode="HTML"` or `parse_mode="Markdown"`.

### `send_text`

```python
from client_utils import send_text
 
 
peer_id = 123456789
 
# Send a plain text message.
send_text(peer_id, "Hello from my plugin!")
 
# Send a reply.
send_text(peer_id, "This is a reply.", replyToMsg=9876)
 
# Send a message with Markdown formatting.
send_text(peer_id, "**Hello** from my *plugin*!", parse_mode="Markdown")
```

### `send_photo`

```python
from client_utils import send_photo
 
 
peer_id = 123456789
photo_path = "/path/to/your/image.jpg"
 
# Send a photo with a caption.
send_photo(peer_id, photo_path, caption="Here is a photo!")
 
# Send a high-quality photo with HTML caption.
send_photo(
    peer_id,
    photo_path,
    caption="<b>High quality</b> photo.",
    high_quality=True,
    parse_mode="HTML",
)
```

### `send_document`

```python
from client_utils import send_document
 
 
peer_id = 123456789
file_path = "/path/to/your/file.zip"
 
send_document(peer_id, file_path, caption="Here is the zip file.")
```

### `send_video`

```python
from client_utils import send_video
 
 
peer_id = 123456789
video_path = "/path/to/your/video.mp4"
 
# The SDK will fill common video attributes such as size and duration.
send_video(peer_id, video_path, caption="Check out this video!")
```

### `send_audio`

```python
from client_utils import send_audio
 
 
peer_id = 123456789
audio_path = "/path/to/your/song.mp3"
 
# The SDK will fill common audio metadata when possible.
send_audio(peer_id, audio_path, caption="Listen to this!")
```

### Low-level `send_message`

If you need more control, use `send_message(params, parse_mode=None)` directly.

It accepts a dictionary of fields that map onto `SendMessagesHelper.SendMessageParams`.

```python
from client_utils import send_message
 
 
send_message(
    {
        "peer": 123456789,
        "message": "Scheduled message from plugin",
        "scheduleDate": 1735689600,
        "notify": False,
    }
)
```

Commonly used keys include:

  * `peer`
  * `message`
  * `caption`
  * `photo`
  * `document`
  * `path`
  * `replyToMsg`
  * `replyMarkup`
  * `params`
  * `notify`
  * `scheduleDate`
  * `ttl`
  * `hasMediaSpoilers`
  * `sendingHighQuality`

> > [!NOTE] **Parse mode** > parse_modeis case-insensitive and currently supports onlyhtmlandmarkdown. When it is provided, the SDK parses eithermessageorcaptionand fillsentitiesfor you.

## Editing Messages

`edit_message(...)` edits an existing `MessageObject`.

You can change:

  * textual content
  * replacement media file
  * spoiler state for replaced media

```python
from client_utils import edit_message
 
 
# Assume `message_obj` is a valid MessageObject instance.
 
# Edit the text of a message.
edit_message(message_obj, text="This is the new, edited text.")
 
# Edit text with HTML formatting.
edit_message(message_obj, text="This is <b>bold</b> text.", parse_mode="HTML")
 
# Replace the media in a message and update the text at the same time.
new_photo_path = "/path/to/another/image.jpg"
edit_message(message_obj, file_path=new_photo_path, text="Here is a new photo instead.")
 
# Replace media with spoiler enabled.
edit_message(message_obj, file_path=new_photo_path, with_spoiler=True)
```

Notes:

  * `message_obj` must be a real `org.telegram.messenger.MessageObject`
  * pass `text=` when you want to replace message text or caption text
  * pass `file_path=` when you want to replace attached media
  * unsupported `parse_mode` values raise `ValueError`

## Fragments, Controllers, and Account Helpers

These helpers return commonly used Telegram components for **the account selected in the UI** , unless you pass an explicit `account=`. See [Multi-account](</docs/multi-account>) — inside a hook, the account you are handling is often not the selected one.

```python
from client_utils import (
    get_last_fragment,
    get_account_instance,
    get_messages_controller,
    get_contacts_controller,
    get_media_data_controller,
    get_connections_manager,
    get_location_controller,
    get_notifications_controller,
    get_messages_storage,
    get_send_messages_helper,
    get_file_loader,
    get_secret_chat_helper,
    get_download_controller,
    get_notifications_settings,
    get_notification_center,
    get_media_controller,
    get_user_config,
)
 
 
fragment = get_last_fragment()
account_instance = get_account_instance()
messages_controller = get_messages_controller()
connections_manager = get_connections_manager()
send_helper = get_send_messages_helper()
user_cfg = get_user_config()
 
if user_cfg.getCurrentUser():
    current_user = user_cfg.getCurrentUser()
 
# Every one of them also takes an explicit account:
messages_controller = get_messages_controller(1)
```

These helpers are thin convenience wrappers, but they keep plugin code much cleaner.

`get_media_controller()` is the one exception with no `account` parameter: `MediaController` is a single global instance, not per-account.

## NotificationCenterDelegate

`NotificationCenterDelegate` is a ready-made Python base class for implementing `NotificationCenter.NotificationCenterDelegate`.

It is useful when you want to listen to client-side notifications from Telegram.

```python
from java.lang import Object
from java import jarray
 
from client_utils import NotificationCenterDelegate, get_notification_center
from org.telegram.messenger import NotificationCenter
 
 
class MyDelegate(NotificationCenterDelegate):
    def didReceivedNotification(self, id: int, account: int, args: jarray(Object)):
        if id == NotificationCenter.didUpdateConnectionState:
            print("Connection state changed:", args)
 
 
delegate = MyDelegate()
notification_center = get_notification_center()
notification_center.addObserver(delegate, NotificationCenter.didUpdateConnectionState)
 
# Later, when you no longer need it:
# notification_center.removeObserver(delegate, NotificationCenter.didUpdateConnectionState)
```

You do not need to wrap `NotificationCenterDelegate` with `dynamic_proxy(...)` yourself. Subclassing it is enough.

If you need a generated Java subclass instead of a simple interface proxy, see [Class Proxy](</docs/class-proxy>).

## Displaying Bulletins

For bulletins and bottom notifications, use `ui.bulletin.BulletinHelper`.

```python
from ui.bulletin import BulletinHelper
 
 
BulletinHelper.show_info("This is an informational message.")
```

For the full list of supported bulletin helpers and examples, see [Bulletin Helper](</docs/bulletin-helper>).

[PreviousAndroid Utilities](</docs/android-utils>)[NextText Formatting](</docs/text-formatting>)


---


# === DOCUMENT: Text Formatting (/docs/text-formatting) ===

---
title: "Text Formatting"
source: "https://plugins.exteragram.app/docs/text-formatting"
---

# Text Formatting

This module provides the ability to parse HTML and Markdown formatted text and convert formatting entities to TLRPC objects suitable for the Telegram API.

The `extera_utils.text_formatting` module allows you to easily convert text with **HTML** or **Markdown** formatting into a plain text string and a list of `TLRPC.MessageEntity` objects.

You generally don't need to use this module directly if you are using `client_utils.send_message` (or `send_text`, etc.), as those functions accept a convenient `parse_mode` parameter. However, if you need to manually parse text for other purposes, this module provides the necessary tools.

## `parse_text`

The main entry point is the `parse_text` function.

```python
from extera_utils.text_formatting import parse_text
 
def parse_text(text: str, parse_mode: Optional[str] = 'HTML', is_caption: bool = False) -> Dict[str, Any]:
    ...
```

**Parameters:**

  * `text`: The input string containing formatting tags or markdown.
  * `parse_mode`: The format to use. Either `'HTML'` (default) or `'Markdown'`.
  * `is_caption`: If `True`, the returned dictionary key for text will be `"caption"`; otherwise `"message"`.

**Returns:** A dictionary containing:

  * `"message"` (or `"caption"`): The plain text content with formatting markers removed.
  * `"entities"`: A list of `TLRPC.MessageEntity` objects.

### Usage Example

```python
from extera_utils.text_formatting import parse_text
from client_utils import send_message
 
# Parsing HTML
html_input = "<b>Bold</b>, <i>Italic</i>, and <a href='https://example.com'>Link</a>"
parsed_html = parse_text(html_input, parse_mode='HTML')
 
# parsed_html is roughly:
# {
#   "message": "Bold, Italic, and Link",
#   "entities": [TL_messageEntityBold(...), TL_messageEntityItalic(...), TL_messageEntityTextUrl(...)]
# }
 
# You can pass this dictionary directly to send_message (via kwargs unpacking if keys match params)
# But client_utils functions handle this automatically via parse_mode argument.
 
# Parsing Markdown
md_input = "**Bold**, __Italic__, and [Link](https://example.com)"
parsed_md = parse_text(md_input, parse_mode='Markdown')
```

## Supported Formatting

### HTML Tags

The HTML parser supports the following tags:

  * `<b>`, `<strong>`: Bold
  * `<i>`, `<em>`: Italic
  * `<u>`: Underline
  * `<s>`, `<del>`, `<strike>`: Strikethrough
  * `<a href="...">`: Text Link
  * `<code>`: Inline Code
  * `<pre language="...">`: Preformatted Code Block (language is optional)
  * `<spoiler>`, `<tg-spoiler>`: Spoiler
  * `<blockquote>`: Blockquote
    * Add attribute `expandable` or `collapsed` for expandable blockquotes (e.g. `<blockquote expandable>`).
  * `<emoji id="...">`: Custom Emoji

### Markdown

The Markdown parser supports the following syntax:

  * `*bold*`: Bold
  * `_italic_`: Italic
  * `__underline__`: Underline
  * `~strikethrough~`: Strikethrough
  * `||spoiler||`: Spoiler
  * ``code``: Inline Code
  * ````code block````: Preformatted Code Block
  * ````python ... ````: Code Block with language
  * `[text](url)`: Text Link
  * `![alt](tg://emoji?id=123)`: Custom Emoji
  * `> Quote`: Blockquote
  * `**> Quote`: Expandable Quote

## Helper Classes

### `TLEntityType`

Enum representing supported entity types: `CODE`, `PRE`, `STRIKETHROUGH`, `TEXT_LINK`, `BOLD`, `ITALIC`, `UNDERLINE`, `SPOILER`, `CUSTOM_EMOJI`, `BLOCKQUOTE`.

### `RawEntity`

Intermediate representation of an entity before converting to TLRPC. Contains offset, length, and extra attributes like `url`, `language`, `document_id` (for emojis), `collapsed` (for blockquotes).

[PreviousClient Utilities](</docs/client-utils>)[NextHook Utilities (Reflection)](</docs/hook-utils>)


---


# === DOCUMENT: File Utilities (/docs/file-utils) ===

---
title: "File Utilities"
source: "https://plugins.exteragram.app/docs/file-utils"
---

# File Utilities

Work with Telegram directories, read and write files, and register custom file open handlers.

The `file_utils` module has two main responsibilities:

  * filesystem helpers for common plugin tasks
  * `FilesController`, which lets you intercept file opening by extension and optionally provide custom file icons

## Standard Directories

These helpers return absolute paths to useful Telegram and plugin directories.

```python
from file_utils import (
    get_plugins_dir,
    get_cache_dir,
    get_files_dir,
    get_images_dir,
    get_videos_dir,
    get_audios_dir,
    get_documents_dir,
)
 
# The directory where plugin files are stored.
plugins_path = get_plugins_dir()
 
# Telegram cache and media directories.
cache_path = get_cache_dir()
files_path = get_files_dir()
images_path = get_images_dir()
videos_path = get_videos_dir()
audios_path = get_audios_dir()
documents_path = get_documents_dir()
```

Available directory helpers:

  * `get_plugins_dir()`
  * `get_cache_dir()`
  * `get_files_dir()`
  * `get_images_dir()`
  * `get_videos_dir()`
  * `get_audios_dir()`
  * `get_documents_dir()`

## Directory Operations

### `ensure_dir_exists(path: str)`

Creates the directory if it does not exist yet, including missing parent directories.

```python
import os
 
from file_utils import ensure_dir_exists, get_plugins_dir
 
# Create a folder for your plugin data if it does not exist yet.
data_dir = os.path.join(get_plugins_dir(), "my_plugin_data")
ensure_dir_exists(data_dir)
```

### `list_dir(...)`

Lists files and or directories with optional recursion and extension filtering.

```python
from file_utils import get_cache_dir, get_images_dir, list_dir
 
# Find image files in Telegram's image directory.
image_files = list_dir(
    path=get_images_dir(),
    extensions=[".jpg", ".png"],
)
 
# Find all cache subdirectories recursively.
cache_dirs = list_dir(
    path=get_cache_dir(),
    recursive=True,
    include_files=False,
    include_dirs=True,
)
```

Parameters:

  * `recursive=False`: walk child directories too
  * `include_files=True`: include files in the result
  * `include_dirs=False`: include directories in the result
  * `extensions=None`: optional suffix filter such as `[".json", ".txt"]`

## File Operations

### `write_file(path: str, content: str)`

Writes text to a file and overwrites any existing content.

```python
import os
 
from file_utils import ensure_dir_exists, get_plugins_dir, write_file
 
# Prepare a plugin-specific folder before writing a file into it.
data_dir = os.path.join(get_plugins_dir(), "my_plugin_data")
ensure_dir_exists(data_dir)
 
# Save a simple text file.
write_file(
    os.path.join(data_dir, "config.txt"),
    "enabled=true",
)
```

### `read_file(path: str)`

Reads the whole file as text.

On failure it returns `None` and logs the exception.

```python
import os
 
from file_utils import get_plugins_dir, read_file
 
# Read the config file back.
config_text = read_file(
    os.path.join(get_plugins_dir(), "my_plugin_data", "config.txt")
)
 
if config_text is not None:
    print(config_text)
```

### `write_file_bytes(path: str, content: bytes)`

Writes binary data to a file.

```python
import os
 
from file_utils import ensure_dir_exists, get_plugins_dir, write_file_bytes
 
# Save raw bytes, for example a downloaded asset.
data_dir = os.path.join(get_plugins_dir(), "my_plugin_data")
ensure_dir_exists(data_dir)
 
write_file_bytes(
    os.path.join(data_dir, "payload.bin"),
    b"\x01\x02\x03",
)
```

### `read_file_bytes(path: str)`

Reads the whole file as bytes.

On failure it returns `None` and logs the exception.

```python
import os
 
from file_utils import get_plugins_dir, read_file_bytes
 
# Read binary data from disk.
payload = read_file_bytes(
    os.path.join(get_plugins_dir(), "my_plugin_data", "payload.bin")
)
 
if payload is not None:
    print(len(payload))
```

### `delete_file(path: str)`

Deletes a file and returns:

  * `True` if the file was deleted
  * `False` if the file did not exist or deletion failed

```python
from file_utils import delete_file
 
# Delete a temporary file if it exists.
was_deleted = delete_file("/path/to/temp.txt")
 
if was_deleted:
    print("The file was removed.")
```

## `FilesController`

`FilesController` lets you register handlers for specific file extensions.

When a user opens a file of a registered type, your callback runs instead of the default open flow.

Typical use cases:

  * open custom file types
  * handle plugin-specific archives
  * launch a custom viewer
  * provide a custom icon for a file extension

> > [!NOTE] **Secrets are required to unregister** > FilesController.register(...)returns a secret string. Keep it somewhere safe if you want to remove the handler later withunregister(...).

## `FilesController.SUPPORT_ICONS`

`FilesController.SUPPORT_ICONS` is `True` when the current client version supports custom file icons.

```python
from file_utils import FilesController
 
# Only try to register custom icons if the client supports them.
if FilesController.SUPPORT_ICONS:
    print("Custom file icons are available.")
```

## `FilesController.Place`

This enum describes where the file open request came from.

Available values:

  * `FilesController.Place.UNKNOWN`
  * `FilesController.Place.ChatActivity`
  * `FilesController.Place.FilteredSearchView`
  * `FilesController.Place.SharedMediaLayout`
  * `FilesController.Place.SearchDownloadsContainer`
  * `FilesController.Place.ChannelAdminLogActivity`

## `FilesController.FileInfo`

Use `FileInfo` to describe a file extension handler.

Fields:

  * `ext: str`
  * `on_click: Callable[[FilesController.OnClickArgs], None]`
  * `whitelist_places: list[FilesController.Place] = []`
  * `blacklist_places: list[FilesController.Place] = []`
  * `get_icon: Optional[Callable[[], Drawable]] = None`

Rules:

  * you cannot use `whitelist_places` and `blacklist_places` together
  * `get_icon` requires `FilesController.SUPPORT_ICONS == True`

### `FilesController.OnClickArgs`

This object is passed to your `on_click` callback.

Fields:

  * `place`
  * `file`
  * `file_name`
  * `message`
  * `activity`
  * `parent_fragment`

## Registering a File Handler

Use `FilesController.register(file_info)` to install a handler.

It returns a secret string which is required for `unregister(...)`.

```python
from file_utils import FilesController
 
 
def on_zip_click(args: FilesController.OnClickArgs):
    # You get the resolved file object and message context here.
    print("Opened:", args.file_name)
    print("From:", args.place)
    print("Absolute path:", args.file.getAbsolutePath())
 
 
# Register a handler for .zip files.
secret = FilesController.register(
    FilesController.FileInfo(
        ext="zip",
        on_click=on_zip_click,
    )
)
```

## Restricting a Handler to Certain Places

Use `whitelist_places` or `blacklist_places` to control where the handler is active.

```python
from file_utils import FilesController
 
 
def on_txt_click(args: FilesController.OnClickArgs):
    # This callback only runs inside chat screens.
    print("Chat text file:", args.file_name)
 
 
secret = FilesController.register(
    FilesController.FileInfo(
        ext="txt",
        on_click=on_txt_click,
        whitelist_places=[FilesController.Place.ChatActivity],
    )
)
```

```python
from file_utils import FilesController
 
 
def on_pdf_click(args: FilesController.OnClickArgs):
    # This callback runs everywhere except the downloads container.
    print("PDF:", args.file_name)
 
 
secret = FilesController.register(
    FilesController.FileInfo(
        ext="pdf",
        on_click=on_pdf_click,
        blacklist_places=[FilesController.Place.SearchDownloadsContainer],
    )
)
```

## Registering a Custom Icon

If icon support is available, you can provide a `Drawable` factory with `get_icon`.

```python
from file_utils import FilesController
 
 
def make_zip_icon():
    # Return an Android Drawable instance here.
    return my_drawable
 
 
def on_zip_click(args: FilesController.OnClickArgs):
    # Handle the file normally when the user taps it.
    print("ZIP opened:", args.file_name)
 
 
if FilesController.SUPPORT_ICONS:
    secret = FilesController.register(
        FilesController.FileInfo(
            ext="zip",
            on_click=on_zip_click,
            get_icon=make_zip_icon,
        )
    )
```

## Unregistering a File Handler

Use the extension and the secret returned by `register(...)`.

```python
from file_utils import FilesController
 
# Remove the handler that was previously registered.
FilesController.unregister("zip", secret)
```

## Exceptions

`FilesController` can raise these exceptions:

  * `FilesController.ExtensionAlreadyRegistered`
  * `FilesController.ExtensionNotRegistered`
  * `FilesController.SecretInvalid`

```python
from file_utils import FilesController
 
try:
    # Register the same extension twice to demonstrate error handling.
    FilesController.register(FilesController.FileInfo(ext="zip", on_click=lambda args: None))
    FilesController.register(FilesController.FileInfo(ext="zip", on_click=lambda args: None))
except FilesController.ExtensionAlreadyRegistered as error:
    print(error)
```

[PreviousHook Utilities (Reflection)](</docs/hook-utils>)[NextIntents](</docs/intents>)


---


# === DOCUMENT: Intents (/docs/intents) ===

---
title: "Intents"
source: "https://plugins.exteragram.app/docs/intents"
---

# Intents

Register global before and after handlers for Telegram links, deeplinks, and custom intents.

The `intents` module provides `IntentsManager`, a global API for reacting to links and intents handled by exteraGram.

This manager is available only on exteraGram `66999 (12.6.4)` and newer.

It lets you register:

  * `before` handlers, called before the original method
  * `after` handlers, called after the original method

## Import

```python
from intents import IntentsManager as IM
```

## `new_global_before_handler(...)`

Registers a global handler that runs in `beforeHookedMethod`.

```python
from intents import IntentsManager as IM
 
 
def on_chat_link(id):
    print("Chat id:", id)
 
 
handle = IM.new_global_before_handler(
    on_chat_link,
    scheme="tg",
    host="chat",
    required_path_args_names=["id"],
)
```

## `new_global_after_handler(...)`

Registers a global handler that runs in `afterHookedMethod`.

```python
from intents import IntentsManager as IM
 
 
def on_after_link(intent=None, scheme=None, host=None):
    print("Handled:", scheme, host)
 
 
handle = IM.new_global_after_handler(
    on_after_link,
    scheme="https",
    host="t.me",
)
```

## Filters

Both registration methods accept the same filter arguments.

### `scheme`

Matches `intent.data.scheme`.

```python
handle = IM.new_global_before_handler(
    lambda: None,
    scheme="tg",
)
```

### `host`

Matches `intent.data.host`.

```python
handle = IM.new_global_before_handler(
    lambda: None,
    scheme="tg",
    host="chat",
)
```

### `path`

Matches `intent.data.path`.

If the value does not start with `/`, the SDK adds it automatically.

```python
def on_profile(path=None):
    print(path)
 
 
handle = IM.new_global_before_handler(
    on_profile,
    scheme="tg",
    host="profile",
    path="/user/42",
)
```

`path` also supports template variables like `{user_id}`.

```python
def on_profile(user_id, path=None):
    print("User:", user_id)
    print("Full path:", path)
 
 
handle = IM.new_global_before_handler(
    on_profile,
    scheme="tg",
    host="profile",
    path="/user/{user_id}",
)
```

### `required_path_args_names`

Despite the name, this currently means required query arguments.

If one of these query arguments is missing, the handler does not run.

```python
def on_chat(id):
    print("Chat:", id)
 
 
handle = IM.new_global_before_handler(
    on_chat,
    scheme="tg",
    host="chat",
    required_path_args_names=["id"],
)
```

This matches `tg://chat?id=123` and does not match `tg://chat`.

### `action`

Matches `intent.getAction()`.

```python
from android.content import Intent
 
 
def on_view_action(action=None):
    print(action)
 
 
handle = IM.new_global_before_handler(
    on_view_action,
    action=Intent.ACTION_VIEW,
)
```

### `whitelist_flags`

Requires every listed flag to be present in `intent.getFlags()`.

```python
from android.content import Intent
 
 
handle = IM.new_global_before_handler(
    lambda: None,
    whitelist_flags=[Intent.FLAG_ACTIVITY_NEW_TASK],
)
```

### `blacklist_flags`

Rejects the intent if any listed flag is present in `intent.getFlags()`.

```python
from android.content import Intent
 
 
handle = IM.new_global_before_handler(
    lambda: None,
    blacklist_flags=[Intent.FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY],
)
```

### `type`

Matches `intent.getType()`.

```python
handle = IM.new_global_before_handler(
    lambda: None,
    type="text/plain",
)
```

### `categories`

Requires every listed category to be present in `intent.getCategories()`.

```python
from android.content import Intent
 
 
handle = IM.new_global_before_handler(
    lambda: None,
    categories=[Intent.CATEGORY_BROWSABLE],
)
```

### `priority`

Handlers are sorted by ascending priority.

Lower numbers run first.

```python
def first():
    print("first")
 
 
def second():
    print("second")
 
 
IM.new_global_before_handler(first, scheme="tg", priority=0)
IM.new_global_before_handler(second, scheme="tg", priority=10)
```

## Callback Arguments

The SDK inspects your callback signature and only passes values that the function explicitly asks for.

Available standard names:

  * `intent`
  * `scheme`
  * `host`
  * `path`
  * `query_args`
  * `action`
  * `flags`
  * `type`
  * `categories`

In addition, the SDK exposes:

  * every query argument by name, such as `id` or `plugin`
  * every path-template variable by name, such as `user_id` from `/user/{user_id}`

### Requesting only one named value

```python
def on_link(host):
    print(host)
 
 
IM.new_global_before_handler(on_link, scheme="tg")
```

### Requesting the full common context

```python
def on_link(intent, scheme, host, path, query_args, action, flags, type, categories):
    print(intent)
    print(scheme)
    print(host)
    print(path)
    print(query_args)
    print(action)
    print(flags)
    print(type)
    print(categories)
 
 
IM.new_global_before_handler(on_link, scheme="https")
```

### Receiving query arguments by name

```python
def on_setting_link(plugin, section=None):
    print("Plugin:", plugin)
    print("Section:", section)
 
 
IM.new_global_after_handler(
    on_setting_link,
    scheme="https",
)
```

### Receiving path variables by name

```python
def on_profile(user_id, path=None):
    print(user_id)
    print(path)
 
 
IM.new_global_before_handler(
    on_profile,
    scheme="tg",
    host="profile",
    path="/user/{user_id}",
)
```

### Using `**kwargs`

If the callback accepts `**kwargs`, the SDK passes the full available named context.

```python
def on_any_link(**kwargs):
    print(kwargs["intent"])
    print(kwargs.get("scheme"))
    print(kwargs.get("id"))
    print(kwargs.get("query_args"))
    print(kwargs.get("flags"))
    print(kwargs.get("categories"))
 
 
IM.new_global_before_handler(on_any_link, scheme="tg")
```

### Using `*args`

If the callback accepts `*args`, the SDK also passes the standard positional bundle:

  * `intent`
  * `scheme`
  * `host`
  * `path`
  * `query_args`
  * `action`
  * `flags`
  * `type`
  * `categories`

```python
def on_any_link(*args):
    intent, scheme, host, path, query_args, action, flags, type, categories = args
    print(scheme, host, path, flags)
 
 
IM.new_global_before_handler(on_any_link, scheme="tg")
```

### Combining named parameters and `**kwargs`

```python
def on_chat(id, intent=None, **kwargs):
    print("Chat id:", id)
    print("Intent:", intent)
    print("Host:", kwargs.get("host"))
    print("Query args:", kwargs.get("query_args"))
 
 
IM.new_global_before_handler(
    on_chat,
    scheme="tg",
    host="chat",
    required_path_args_names=["id"],
)
```

## Stopping Further Handling

`before` handlers can stop the rest of the Python handlers and also stop the original Java intent handling.

To do that, return `True` from the callback.

Behavior:

  * only `before` handlers can stop handling
  * once a `before` callback returns `True`, later handlers are not called
  * the SDK returns `true` from the Xposed hook by calling `param.setResult(true)`

```python
def intercept_chat(id):
    print("Intercepted:", id)
    return True
 
 
IM.new_global_before_handler(
    intercept_chat,
    scheme="tg",
    host="chat",
    required_path_args_names=["id"],
)
```

Returning any value other than `True` does not stop handling.

`after` handlers cannot stop the original intent flow.

## `HandlerHandle`

Both registration methods return a `HandlerHandle`.

Call `unhandle()` on it to remove the registered handler.

```python
from intents import IntentsManager as IM
 
 
def on_link():
    print("called")
 
 
handle = IM.new_global_before_handler(on_link, scheme="tg")
handle.unhandle()
```

## `unhandle(handler_id)`

You can also remove a handler manually by its id.

```python
from intents import IntentsManager as IM
 
 
def on_link():
    print("called")
 
 
handle = IM.new_global_before_handler(on_link, scheme="tg")
IM.unhandle(handle.handler_id)
```

## `parse(url: str)`

`parse(...)` is a small helper for parsing URL-like strings.

It returns a dictionary with at least:

  * `scheme`
  * `host`

It may also include:

  * `path`
  * `query`

```python
from intents import IntentsManager as IM
 
parts = IM.parse("tg://chat?id=123")
 
print(parts["scheme"])  # "tg"
print(parts["host"])    # "chat"
print(parts["query"])   # "id=123"
```

## Exceptions

`IM.unhandle(...)` raises `IntentsManager.HandlerNotRegistered` if the handler id does not exist.

```python
from intents import IntentsManager as IM
 
try:
    IM.unhandle("missing-handler-id")
except IM.HandlerNotRegistered as error:
    print(error)
```

## Notes

  * matching by `scheme`, `host`, `path`, required query arguments, `action`, `flags`, `type`, and `categories` happens on the Java side
  * only matching handler ids are dispatched to Python
  * handlers are executed in ascending `priority` order
  * `required_path_args_names` currently means required query-argument names
  * `before` handlers may stop both later handlers and the original Java intent handling by returning `True`

[PreviousFile Utilities](</docs/file-utils>)[NextAlert Dialog Builder](</docs/alert-dialog-builder>)


---


# === DOCUMENT: Alert Dialog Builder (/docs/alert-dialog-builder) ===

---
title: "Alert Dialog Builder"
source: "https://plugins.exteragram.app/docs/alert-dialog-builder"
---

# Alert Dialog Builder

A Pythonic wrapper for creating and managing Telegram-style AlertDialogs.

The `AlertDialogBuilder` class, found in `ui/alert`, provides a convenient way to construct and display various types of alert dialogs within your plugins. It wraps `org.telegram.ui.ActionBar.AlertDialog.Builder` and simplifies its usage from Python.

## Basic Usage

```python
from ui.alert import AlertDialogBuilder
from client_utils import get_last_fragment
from android_utils import log
 
# Get current activity (context)
current_fragment = get_last_fragment()
if not current_fragment:
    log("Cannot show dialog, no current fragment.")
    # return or handle error
 
activity = current_fragment.getParentActivity()
if not activity:
    log("Cannot show dialog, no parent activity.")
    # return or handle error
 
# Create a simple message dialog
builder = AlertDialogBuilder(activity) # Default is ALERT_TYPE_MESSAGE
builder.set_title("My Plugin Alert")
builder.set_message("This is an important message from the plugin.")
 
# Add buttons
def on_positive_click(bld: AlertDialogBuilder, which: int):
    log("Positive button clicked!")
    bld.dismiss()
 
def on_negative_click(bld: AlertDialogBuilder, which: int):
    log("Negative button clicked!")
    bld.dismiss()
 
builder.set_positive_button("OK", on_positive_click)
builder.set_negative_button("Cancel", on_negative_click)
 
# create() is optional, but useful if you want to tweak flags before show()
builder.create()
builder.set_cancelable(True)
builder.show()
```

## Dialog Types

`AlertDialogBuilder` supports different styles of dialogs, controlled by the `progress_style` parameter in its constructor:

  * `AlertDialogBuilder.ALERT_TYPE_MESSAGE` (default): Standard message dialog.
  * `AlertDialogBuilder.ALERT_TYPE_LOADING`: Dialog with a determinate horizontal progress bar. Use `builder.set_progress(value)` to update.
  * `AlertDialogBuilder.ALERT_TYPE_SPINNER`: Dialog with an indeterminate spinner, often used for loading states.

```python
# Loading dialog example
loading_builder = AlertDialogBuilder(activity, AlertDialogBuilder.ALERT_TYPE_SPINNER)
loading_builder.set_title("Loading Data...")
loading_builder.set_message("Please wait while data is being fetched.")
loading_builder.show()
loading_builder.set_cancelable(False) # Best called after create() or show()
 
# Later, when loading is done:
# loading_builder.dismiss()
```

## Key Methods

### Initialization

  * `AlertDialogBuilder(context: Context, progress_style: int = ALERT_TYPE_MESSAGE, resources_provider: Optional[Theme.ResourcesProvider] = None)`: Constructor.

### Content

  * `set_title(title: str)`: Sets the dialog title.
  * `set_message(message: str)`: Sets the main message content.
  * `set_message_text_view_clickable(clickable: bool)`: Makes the message text clickable (e.g., for links).
  * `set_view(view: View, height: int = -2)`: Sets a custom Android `View` as the dialog's content.
  * `set_items(items: List[str], listener: Optional[Callable[['AlertDialogBuilder', int], None]] = None, icons: Optional[List[int]] = None)`: Displays a list of items. The listener is called with the dialog builder instance and the index of the clicked item.

### Buttons

  * `set_positive_button(text: str, listener: Optional[Callable[['AlertDialogBuilder', int], None]] = None)`
  * `set_negative_button(text: str, listener: Optional[Callable[['AlertDialogBuilder', int], None]] = None)`
  * `set_neutral_button(text: str, listener: Optional[Callable[['AlertDialogBuilder', int], None]] = None)`
    * Listeners receive the `AlertDialogBuilder` instance and a button identifier (`AlertDialogBuilder.BUTTON_POSITIVE`, etc.).
  * `make_button_red(button_type: int)`: Styles a button's text (e.g., `AlertDialogBuilder.BUTTON_NEGATIVE`) with red color (using `Theme.key_text_RedBold`).

### Listeners

  * `set_on_back_button_listener(listener: Optional[Callable[['AlertDialogBuilder', int], None]] = None)`: For back button presses while the dialog is shown.
  * `set_on_dismiss_listener(listener: Optional[Callable[['AlertDialogBuilder'], None]] = None)`: Called when the dialog is dismissed for any reason.
  * `set_on_cancel_listener(listener: Optional[Callable[['AlertDialogBuilder'], None]] = None)`: Called when the dialog is cancelled (e.g., by back press or touch outside, if cancelable).

### Appearance & Behavior

  * `set_top_image(res_id: int, background_color: int)`
  * `set_top_drawable(drawable: Drawable, background_color: int)`
  * `set_top_animation(res_id: int, size: int, auto_repeat: bool, background_color: int, layer_colors: Optional[Dict[str, int]] = None)`
  * `set_dim_enabled(enabled: bool)`: Enables/disables dimming of the background.
  * `set_dialog_button_color_key(theme_key: int)`: Sets a theme color key for buttons.
  * `set_blurred_background(blur: bool, blur_behind_if_possible: bool = True)`: Attempts to apply a blurred background.
  * `set_cancelable(cancelable: bool)`: Sets if the dialog can be dismissed by tapping outside or pressing back. Best called _after_ `create()` or `show()`.
  * `set_canceled_on_touch_outside(cancel: bool)`: Sets if tapping outside dismisses. Best called _after_ `create()` or `show()`.

### Lifecycle

  * `create() -> 'AlertDialogBuilder'`: Creates the dialog but doesn't show it.
  * `show() -> 'AlertDialogBuilder'`: Creates (if not already) and shows the dialog.
  * `dismiss()`: Dismisses the dialog if it's showing.
  * `get_dialog() -> Optional[AlertDialog]`: Returns the underlying Java `AlertDialog` instance, or `None` once it has been dismissed and released.
  * `get_button(button_type: int) -> Optional[View]`: Gets a button view from the dialog (e.g., for custom styling). Call after `create()` or `show()`.

### Progress

  * `set_progress(progress: int)`: Sets the progress for `ALERT_TYPE_LOADING` dialogs (0-100).

## Example: Dialog with Items

```python
from ui.alert import AlertDialogBuilder
from client_utils import get_last_fragment
from android_utils import log
 
def on_item_click(bld: AlertDialogBuilder, which: int):
    items_list = ["Option A", "Option B", "Option C"]
    log(f"Item '{items_list[which]}' (index {which}) selected.")
    bld.dismiss()
 
item_builder = AlertDialogBuilder(activity)
item_builder.set_title("Choose an Option")
item_builder.set_items(
    ["Option A", "Option B", "Option C"],
    on_item_click
)
item_builder.set_negative_button("Cancel", lambda b, w: b.dismiss())
item_builder.show()
```

## Important Notes

  * **Context** : Always provide a valid Android `Context` (usually an `Activity`) to the constructor. `get_last_fragment().getParentActivity()` is a common way to get this.
  * **Listeners** : The listener callables you provide will receive the Python `AlertDialogBuilder` instance as their first argument, allowing you to interact with the dialog (e.g., `bld.dismiss()`) from within the callback.
  * **Thread Safety** : Dialog manipulation (creating, showing, dismissing, updating content) should generally happen on the Android UI thread. Use `android_utils.run_on_ui_thread` if you're performing these actions from a background thread.
  * **Error Handling** : The proxy listeners in `alert.py` include basic try-except blocks to log errors occurring within your Python callbacks, preventing crashes.
  * **One dialog per builder** : Once the dialog is dismissed, the builder only refers to it weakly, so the dialog, and everything your listeners use, can be freed. A dismissed dialog can't be shown again: create a new `AlertDialogBuilder` each time you show one.

[PreviousIntents](</docs/intents>)[NextBulletin Helper](</docs/bulletin-helper>)


---


# === DOCUMENT: Bulletin Helper (/docs/bulletin-helper) ===

---
title: "Bulletin Helper"
source: "https://plugins.exteragram.app/docs/bulletin-helper"
---

# Bulletin Helper

Easily display various types of bottom-screen notifications (Bulletins) in your plugins.

The `BulletinHelper` class, found in `bulletin.py`, provides a set of static methods to conveniently show Telegram's "Bulletin" notifications. Bulletins are small, non-intrusive messages that typically appear at the bottom of the screen and dismiss automatically.

If you do not pass a fragment explicitly, the helper tries a few sensible fallbacks:

  * the fragment you passed in
  * the current safe fragment from `LaunchActivity`
  * the visible bulletin bottom sheet, if one is active
  * the global bulletin factory as a last resort

## Basic Usage

Most `BulletinHelper` methods are class methods and can be called directly. They often accept an optional `fragment` argument; if not provided, the helper tries to use the currently active fragment or a global context.

```python
from ui.bulletin import BulletinHelper
from client_utils import get_last_fragment # Optional, for explicit fragment passing
from org.telegram.messenger import R as R_tg # For Telegram's R.raw Lottie animations
 
# Get current fragment (optional)
current_fragment = get_last_fragment()
 
# Show a simple informational bulletin
BulletinHelper.show_info("This is some information.", current_fragment)
 
# Show an error bulletin
BulletinHelper.show_error("An error occurred processing your request.", current_fragment)
 
# Show a success bulletin
BulletinHelper.show_success("Action completed successfully!", current_fragment)
```

> > [!NOTE] **UI Thread** > AllBulletinHelper.show_...methods automatically ensure that the bulletin is shown on the Android UI thread, so you don't need to wrap these calls inrun_on_ui_threadyourself.

## Bulletin Types and Methods

`BulletinHelper` wraps common functionalities of `org.telegram.ui.Components.BulletinFactory`.

### Standard Bulletins

  * `BulletinHelper.show_info(message: str, fragment: Optional[BaseFragment] = None)`
    * Shows a bulletin with a default info icon (e.g., `R.raw.info`).
  * `BulletinHelper.show_error(message: str, fragment: Optional[BaseFragment] = None)`
    * Shows a bulletin with a default error/alert icon.
  * `BulletinHelper.show_success(message: str, fragment: Optional[BaseFragment] = None)`
    * Shows a bulletin with a default success/check icon.

### Custom Simple Bulletins

  * `BulletinHelper.show_simple(text: str, icon_res_id: int, fragment: Optional[BaseFragment] = None)`
    * Shows a single-line bulletin with a custom Lottie animation icon.
    * `icon_res_id`: A Lottie animation resource ID (e.g., `R_tg.raw.some_animation`).

```python
BulletinHelper.show_simple("Processing...", R_tg.raw.timer, current_fragment)
```

  * `BulletinHelper.show_two_line(title: str, subtitle: str, icon_res_id: int, fragment: Optional[BaseFragment] = None)`
    * Shows a two-line bulletin with a custom icon, title, and subtitle.

```python
BulletinHelper.show_two_line("Download Complete", "File saved to gallery.", R_tg.raw.ic_download_done, current_fragment)
```

### Bulletins with Actions

  * `BulletinHelper.show_with_button(text: str, icon_res_id: int, button_text: str, on_click: Optional[Callable[[], None]], fragment: Optional[BaseFragment] = None, duration: int = BulletinHelper.DURATION_PROLONG)`

    * Shows a bulletin with an icon, text, and a clickable button.
    * `on_click`: A callable to execute when the button is pressed.
    * `duration`: How long the bulletin stays visible (e.g., `BulletinHelper.DURATION_SHORT`, `DURATION_LONG`, `DURATION_PROLONG`).

```python
def open_settings_action():
    # Code to open some settings page
    print("Settings button clicked!")
 
BulletinHelper.show_with_button(
    "Plugin settings updated.",
    R_tg.raw.info,
    "Configure",
    open_settings_action,
    current_fragment
)
```

  * `BulletinHelper.show_undo(text: str, on_undo: Callable[[], None], on_action: Optional[Callable[[], None]] = None, subtitle: Optional[str] = None, fragment: Optional[BaseFragment] = None)`

    * Shows an "Undo"-style bulletin.
    * `on_undo`: Called if the "Undo" button is pressed.
    * `on_action`: Called after a delay if "Undo" is _not_ pressed (e.g., to commit an action).

```python
def perform_delete():
    print("Item permanently deleted.")
 
def undo_delete():
    print("Delete operation undone.")
 
BulletinHelper.show_undo(
    "Item moved to trash.",
    on_undo=undo_delete,
    on_action=perform_delete,
    fragment=current_fragment
)
```

### Contextual Bulletins (Predefined)

  * `BulletinHelper.show_copied_to_clipboard(message: Optional[str] = None, fragment: Optional[BaseFragment] = None)`
    * Shows "Text copied to clipboard" or a custom message.
  * `BulletinHelper.show_link_copied(is_private_link_info: bool = False, fragment: Optional[BaseFragment] = None)`
    * Shows "Link copied" bulletin, with a variant for private link info.
  * `BulletinHelper.show_file_saved_to_gallery(is_video: bool = False, amount: int = 1, fragment: Optional[BaseFragment] = None)`
    * Shows "Photo/Video saved to gallery" (or plural versions).
  * `BulletinHelper.show_file_saved_to_downloads(file_type_enum_name: str = "UNKNOWN", amount: int = 1, fragment: Optional[BaseFragment] = None)`
    * Shows "File saved to downloads" or similar, based on `BulletinFactory.FileType`.
    * `file_type_enum_name`: String name of the enum from `BulletinFactory.FileType` (e.g., `"PHOTO_TO_DOWNLOADS"`, `"GIF"`).

```python
BulletinHelper.show_file_saved_to_downloads("MUSIC", amount=3, fragment=current_fragment)
```

## Durations

The `BulletinHelper` class defines constants for common durations:

  * `BulletinHelper.DURATION_SHORT` (1500 ms)
  * `BulletinHelper.DURATION_LONG` (2750 ms)
  * `BulletinHelper.DURATION_PROLONG` (5000 ms)

These can be used with methods like `show_with_button`.

## Finding Lottie Animations (`R.raw...`)

Lottie animations used for bulletin icons are typically stored as raw resources in Telegram's codebase. You can explore Telegram's source (specifically `TMessagesProj/src/main/res/raw/`) to find available animations (e.g., `info.json`, `success.json`, `delete.json`). In Python, these are accessed via `org.telegram.messenger.R.raw.animation_name` (e.g., `R_tg.raw.info`).

[PreviousAlert Dialog Builder](</docs/alert-dialog-builder>)[NextPlugin Repositories](</docs/plugin-repositories>)


---


# === DOCUMENT: Elyx Overview (/docs/elyx) ===

---
title: "Elyx Overview"
source: "https://plugins.exteragram.app/docs/elyx"
---

# Elyx

Build structured, multi-file exteraGram plugins with resources, localization, dependencies, and live reload.

# Elyx structured plugins

Elyx is the structured plugin format for exteraGram. It keeps the familiar [`BasePlugin`](</docs/plugin-class>) API, but removes the requirement to put an entire project into one Python file.

An Elyx plugin can contain:

  * any number of Python modules and packages
  * images, SVG files, Lottie animations, JSON, YAML, and other resources
  * separate localization files
  * bundled Python wheels
  * metadata and path configuration outside the source code
  * a desktop development workflow which reloads changes on a connected phone

The installed plugin is still a normal `BasePlugin` subclass. Elyx is responsible for finding its files, creating a plugin-specific environment, loading dependencies, and isolating local imports.

## When to use Elyx

Use a regular single-file plugin when the complete implementation is small and has no project resources. Use Elyx when at least one of these is true:

  * the source has naturally grown into several modules
  * UI code needs bundled images, animations, or configuration files
  * text should be translated without editing Python code
  * the plugin needs local libraries or a repeatable build
  * several people work on the plugin
  * fast file synchronization is more convenient than reinstalling a `.py` file

## Mental model

An Elyx archive is a ZIP-compatible file with an `.elyx` or `.eaf` extension. Its root contains a `refmap` file which points the runtime at the rest of the project.

```python
my_plugin.elyx
|-- refmap.yml
|-- main.py
|-- metainfo.yml
|-- src/
|   |-- __init__.py
|   `-- feature.py
|-- assets/
|   `-- logo.svg
`-- strings/
    |-- strings_en.yml
    `-- strings_ru.yml
```

At runtime the plugin imports its environment from one stable public module:

```bash
from elyx import assets, metainfo, refmap, settings, strings
```

Plugin authors should import only from `elyx`. Runtime implementation modules are private and may change without compatibility guarantees.

## Documentation map

### [Quick startCreate, install, and run a complete multi-file plugin.](</docs/elyx/quick-start>)### [Project structureLearn the archive layout and every supported `refmap` key.](</docs/elyx/project-structure>)### [ElyxBuilderGenerate a project and build source or compiled archives from the CLI.](</docs/elyx/elyxbuilder>)### [Modules and importsSplit code safely and load Python, JSON, YAML, and text modules.](</docs/elyx/modules-and-imports>)### [AssetsWork with images, SVG, Lottie, bytes, and structured resource files.](</docs/elyx/assets>)### [LocalizationStore translations separately and format localized values.](</docs/elyx/localization>)### [Public APIReference every supported symbol exported by `elyx`.](</docs/elyx/public-api>)

## Elyx and the regular SDK

Elyx does not replace the rest of the plugin SDK. Lifecycle methods, hooks, menu items, Telegram requests, and settings rows still come from the existing SDK:

  * [`BasePlugin` and lifecycle](</docs/plugin-class>)
  * [Plugin settings UI](</docs/plugin-settings>)
  * [Client utilities](</docs/client-utils>)
  * [Java class proxy](</docs/class-proxy>)
  * [Xposed-style hooking](</docs/xposed-hooking>)

Elyx adds project structure and a stable plugin environment around those APIs.

[PreviousFirst Plugin](</docs/first-plugin>)[NextQuick Start](</docs/elyx/quick-start>)


---


# === DOCUMENT: Elyx Quick Start (/docs/elyx/quick-start) ===

---
title: "Elyx Quick Start"
source: "https://plugins.exteragram.app/docs/elyx/quick-start"
---

[Elyx](</docs/elyx>)

# Quick Start

Create and install a complete Elyx plugin.

# Build your first Elyx plugin

This tutorial creates a small localized greeting plugin with multiple Python files, persistent settings, and an SVG asset.

## Create with ElyxBuilder

The fastest way to start is to let ElyxBuilder generate the initial project:

```bash
python -m pip install --upgrade ElyxBuilder
mkdir elyx_hello
cd elyx_hello
elyb new
```

Complete the interactive wizard, then edit the generated metadata, source, locales, and resources. To generate a project without prompts:

```python
elyb new -g -n "Elyx Hello" -a yourname
```

The following steps build the same concepts manually, which is useful for understanding `refmap` and the archive layout. If you started with ElyxBuilder, keep its generated paths and adapt the example code to `<GeneratedName>/src/main.py`.

See the complete [ElyxBuilder guide](</docs/elyx/elyxbuilder>) for project generation, compiled builds, watch mode, ignore lists, and links to the official CLI documentation.

## 1\. Create the project

Create this directory tree:

```python
elyx_hello/
|-- refmap.yml
|-- metainfo.yml
|-- main.py
|-- src/
|   |-- __init__.py
|   `-- greeting.py
|-- assets/
|   `-- wave.svg
`-- strings/
    |-- strings_en.yml
    `-- strings_ru.yml
```

The directory name is not used as plugin metadata during installation, but keeping it equal to `id` makes local development easier.

## 2\. Map the files

Create `refmap.yml`:

```python
metainfo: metainfo.yml
main: main.py
assets: assets
strings: strings
```

All paths are relative to the archive root.

## 3\. Add metadata

Create `metainfo.yml`:

```python
id: elyx_hello
name: Elyx Hello
description: "{plugin_description}"
author: Your Name
version: "1.0.0"
icon: exteraPlugins/1
app_version: ">=12.5.1"
sdk_version: ">=1.4.5.3"
```

`id` and `name` are required. The `{plugin_description}` placeholder is read from the current locale.

## 4\. Add translations

Create `strings/strings_en.yml`:

```python
plugin_description: A small structured Elyx plugin
loaded: Elyx Hello loaded
greeting: "Hello, {name}!"
settings_title: Greeting
name_label: Name
name_hint: Who should be greeted?
```

Create `strings/strings_ru.yml`:

```python
plugin_description: Небольшой структурированный плагин Elyx
loaded: Elyx Hello загружен
greeting: "Привет, {name}!"
settings_title: Приветствие
name_label: Имя
name_hint: Кого нужно поприветствовать?
```

English is the fallback locale, so always ship an `en` file.

## 5\. Split out a helper module

Create `src/__init__.py` as an empty file, then add `src/greeting.py`:

```bash
from elyx import strings
 
 
def build_greeting(name: str) -> str:
    return strings("greeting", name=name)
```

Local modules are isolated inside this plugin. Another installed plugin can also have `src/greeting.py` without causing an import collision.

## 6\. Create the plugin class

Create `main.py`:

```bash
from typing import Any, List
 
from base_plugin import BasePlugin
from elyx import assets, settings, strings
from src.greeting import build_greeting
from ui.settings import Header, Input, Text
 
 
class ElyxHelloPlugin(BasePlugin):
    def on_plugin_load(self):
        self.logger.info(strings("loaded"))
 
        # Access by normalized stem, without the .svg extension.
        wave_icon = assets.wave
        self.logger.info(f"Bundled icon: {wave_icon.path_str}")
 
    def create_settings(self) -> List[Any]:
        name = settings.get("name", "Alice")
        return [
            Header(text=strings("settings_title")),
            Input(
                key="name",
                text=strings("name_label"),
                subtext=strings("name_hint"),
                default="Alice",
                icon="msg_edit",
            ),
            Text(
                text=build_greeting(name),
                subtext="This row is generated from two project files.",
                icon="msg_info",
            ),
        ]
```

There are no top-level `__id__` or `__name__` constants in `main.py`. Structured plugin metadata comes from `metainfo.yml`.

## 7\. Add an asset

Save any valid SVG as `assets/wave.svg`. A minimal example is:

```python
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24">
  <path fill="#4A90E2" d="M7 11V4a1 1 0 0 1 2 0v5h1V3a1 1 0 0 1 2 0v6h1V4a1 1 0 0 1 2 0v6h1V6a1 1 0 0 1 2 0v7c0 5-3 8-8 8s-7-3-7-7v-3a2 2 0 0 1 4 0Z"/>
</svg>
```

## 8\. Create an archive

ZIP the **contents** of `elyx_hello`, not the outer directory. The first level inside the archive must contain `refmap.yml`.

```python
correct:   refmap.yml
incorrect: elyx_hello/refmap.yml
```

Rename the resulting ZIP file to `elyx_hello.elyx` or `elyx_hello.eaf`.

If you use ElyxBuilder, run the command from the directory containing `refmap.yml`:

```python
elyb build --ast --verbose --no-folder
```

See [ElyxBuilder](</docs/elyx/elyxbuilder>) for compiled builds and watch mode, and [Development and build](</docs/elyx/development>) for live reload.

## 9\. Install and enable it

  1. Send the archive to the device or open it from a file manager.
  2. Open it with exteraGram.
  3. Review the metadata and installation warnings.
  4. Install the plugin and enable it.
  5. Open the plugin settings page.

The greeting row should follow the device language and the name saved in the input row.

## What to read next

  * [Project structure](</docs/elyx/project-structure>)
  * [ElyxBuilder](</docs/elyx/elyxbuilder>)
  * [Metadata](</docs/elyx/metadata>)
  * [Modules and imports](</docs/elyx/modules-and-imports>)
  * [Assets](</docs/elyx/assets>)
  * [Localization](</docs/elyx/localization>)

[PreviousElyx](</docs/elyx>)[NextElyxBuilder](</docs/elyx/elyxbuilder>)


---


# === DOCUMENT: ElyxBuilder CLI (/docs/elyx/elyxbuilder) ===

---
title: "ElyxBuilder CLI"
source: "https://plugins.exteragram.app/docs/elyx/elyxbuilder"
---

[Elyx](</docs/elyx>)

# ElyxBuilder

Scaffold, validate, build, watch, and inspect Elyx plugin projects from the command line.

# ElyxBuilder

[ElyxBuilder](<https://github.com/shareui/ElyxBuilder>) is the command-line tool for creating and packaging Elyx projects. It can generate the initial structure, validate Python syntax, build source or compiled archives, rebuild when files change, and maintain build ignore lists.

ElyxBuilder is optional. Elyx does not require it at runtime, and a correctly structured ZIP archive can still be built manually.

## Install the CLI

ElyxBuilder requires Python 3.10 or newer:

```bash
python -m pip install --upgrade ElyxBuilder
```

Verify the installation and inspect the commands supported by the installed version:

```python
elyb --version
elyb --help
```

Python 3.11 is required only when creating compiled `.pyc` builds, because the bytecode version must match the Python runtime on the device.

## Create a project interactively

Open an empty directory which will become the repository root and run:

```python
elyb new
```

The interactive wizard asks for the plugin name, author, id, version constraints, optional icon, and the `refmap` and metadata formats. Press Enter to accept a generated value.

The generated project resembles:

```python
my_plugin_repository/
|-- refmap.yml
`-- MyPlugin/
    |-- .elyxbuilder/
    |   `-- config.yml
    |-- locales/
    |   |-- strings_en.json
    |   `-- strings_ru.json
    |-- meta.yml
    |-- res/
    |   `-- .gitkeep
    `-- src/
        `-- main.py
```

`refmap.yml` stays in the repository and archive root. The other files live in a normalized directory based on the plugin name. ElyxBuilder writes all paths into `refmap`, so plugin authors do not need to move the generated files.

Before the first build:

  1. replace the generated descriptions in `locales`
  2. review the id and version constraints in `meta.yml`
  3. implement the plugin in `src/main.py` and additional modules
  4. add resources to `res`

## Create a project without prompts

Use `--gen` in scripts or when all required values are already known:

```python
elyb new --gen --name "My Plugin" --author myname
```

The short form is equivalent:

```python
elyb new -g -n "My Plugin" -a myname
```

Choose a different archive extension with `--zipformat`:

```python
elyb new -g -n "My Plugin" -a myname --zipformat elyx
```

In generation mode, `--name` and `--author` are required. ElyxBuilder normalizes the directory name and creates the plugin id as `author_PluginName`, limited to 32 characters. The display name in metadata is kept unchanged.

## Build the plugin

Run build commands from the directory containing `refmap.yml`.

For a source build with syntax validation:

```python
elyb build --ast --verbose --no-folder
```

For a compiled build optimized at level 2:

```python
elyb build --compile 2 --verbose --no-folder
```

The shorter form used by many Elyx projects is:

```python
elyb build -c 2 -v -nf
```

Builds are written to `builds/`. `--no-folder` excludes the `.elyxbuilder` directory from the published archive. `--ast` and `--compile` cannot be used together because compilation already parses the Python files.

Useful build options include:

Option| Purpose  
---|---  
`-v`, `--verbose`| Print the complete build log  
`-a`, `--ast`| Validate Python files before packaging  
`-c [LEVEL]`, `--compile [LEVEL]`| Compile Python using optimization level 0, 1, or 2  
`-r`, `--reset`| Clear the incremental compilation cache  
`-nf`, `--no-folder`| Exclude the ElyxBuilder configuration directory  
`--no-assets`| Exclude resources listed in `optionalAssets`  
`-ni`, `--no-info`| Omit ElyxBuilder build metadata  
`-sv VERSION [APPEND]`| Record a target version and optionally append it to the archive name  
`-sc PACKAGE [NAME]`| Record a target client and optionally append its name  
  
Run `elyb build --help` before relying on advanced options in automation. The installed version is the source of truth for its supported arguments.

## Rebuild when files change

`watch` polls the project and starts a build after a change:

```python
elyb watch 10 --args "--ast -v -nf"
```

The first argument is the polling interval in seconds. ElyxBuilder stores the most recent result as `builds/latest.eaf`, or with the archive extension configured for the project.

In the interactive watch screen:

  * press `P` to pause or resume
  * press `Q` to exit

`elyb watch` rebuilds the archive but does not install it on a device. Use the custom `elyx_dev_client.py` workflow for synchronizing individual changes and reloading an installed plugin on the phone.

## Compilation cache

Compiled builds use an incremental cache. Check which source files changed since the previous compiled build:

```python
elyb cached
```

Use `elyb build --compile --reset` when a clean compilation is required.

## Manage ignored files

ElyxBuilder keeps ignore lists in `.elyxbuilder/config.yml`. Prefer the CLI when updating them:

```python
elyb add-ignore "MyPlugin/generated/*" --all
elyb add-ignore "MyPlugin/res/preview.psd" --no-assets
elyb add-ignore "MyPlugin/src/compat.py" --compile
```

The lists have different meanings:

Target| Configuration key| Effect  
---|---|---  
`--all`| `ignoreAll`| Exclude the path from every build  
`--no-assets`| `optionalAssets`| Exclude it only when building with `--no-assets`  
`--compile`| `compilationIgnore`| Keep matching Python files as source  
  
Remove an entry by its zero-based list index:

```python
elyb del-ignore 0 --all
```

## Project statistics

The CLI can inspect the project without building it:

```python
elyb stats builds
elyb stats lines
elyb stats size
elyb stats files
```

Add `--all` to include the complete plugin rather than only the default source scope. `--additional DIR...` can include other repository directories when used together with `--all`.

## Official ElyxBuilder documentation

For every flag and advanced feature, see the upstream documentation:

  * [ElyxBuilder documentation in Russian](<https://github.com/shareui/ElyxBuilder/tree/main/docs/ru>)
  * [ElyxBuilder documentation in English](<https://github.com/shareui/ElyxBuilder/blob/main/docs/en/en.md>)
  * [ElyxBuilder releases](<https://github.com/shareui/ElyxBuilder/releases>)

[PreviousQuick Start](</docs/elyx/quick-start>)[NextProject Structure](</docs/elyx/project-structure>)


---


# === DOCUMENT: Elyx Project Structure (/docs/elyx/project-structure) ===

---
title: "Elyx Project Structure"
source: "https://plugins.exteragram.app/docs/elyx/project-structure"
---

[Elyx](</docs/elyx>)

# Project Structure

Understand Elyx directories, archives, refmap files, and entry points.

# Project structure and `refmap`

Elyx deliberately separates physical layout from runtime meaning. You may organize a small plugin at the root:

```python
my_plugin/
|-- refmap.yml
|-- metainfo.yml
|-- main.py
|-- assets/
`-- strings/
```

or keep distributable files under a nested directory:

```python
my_plugin/
|-- refmap.yml
|-- plugin/
|   |-- meta.yml
|   |-- src/
|   |   |-- __init__.py
|   |   `-- main.py
|   |-- locales/
|   `-- res/
`-- wheels/
```

Both layouts work when `refmap` points to the correct locations.

## Supported `refmap` formats

At the archive root, Elyx discovers the first matching file:

  * `refmap.yaml`
  * `refmap.yml`
  * `refmap.json`

YAML is recommended because it is concise and contains no executable code.

```python
metainfo: plugin/meta.yml
main: plugin/src/main.py
strings: plugin/locales
assets: plugin/res
wheels: wheels
```

The equivalent JSON is:

```json
{
  "metainfo": "plugin/meta.yml",
  "main": "plugin/src/main.py",
  "strings": "plugin/locales",
  "assets": "plugin/res",
  "wheels": "wheels"
}
```

## Runtime keys

### `main`

Path to the Python entry module. The module must contain at least one class which inherits from `BasePlugin`.

```python
main: plugin/src/main.py
```

If omitted, Elyx uses `main.py` at the archive root when it exists.

Only the first `BasePlugin` subclass found in the entry module is instantiated. Keep one plugin class in the entry module and import helper classes from other files.

### `metainfo`

Path to a YAML, JSON, or Python metadata file:

```python
metainfo: plugin/meta.yml
```

If omitted, Elyx searches the archive root for `metainfo.yaml`, `metainfo.yml`, or `metainfo.json`.

### `assets`

Path to the root resource directory:

```python
assets: plugin/res
```

When present and non-empty, plugin modules can use:

```bash
from elyx import assets
```

If the key is omitted, Elyx automatically uses a root-level `assets` directory when present.

### `strings`

Path to either one localization file or a directory of locale files:

```python
strings: plugin/locales
```

See [Localization](</docs/elyx/localization>) for file naming and fallback rules.

### `wheels`

Path to a directory containing bundled `.whl` files:

```python
wheels: wheels
```

See [Dependencies](</docs/elyx/dependencies>) before bundling wheels.

## Builder-only keys

Tools may add their own values to `refmap`, for example:

```python
elyxbuilder: plugin/.elyxbuilder
```

The current runtime ignores unknown keys. They remain available to plugin code through:

```bash
from elyx import refmap
```

This makes `refmap` extensible without exposing engine internals.

## Archive rules

Elyx archives are ZIP-compatible and normally use one of these extensions:

  * `.elyx`
  * `.eaf`
  * `.elyx.zip`
  * `.eaf.zip`

The installable archive must contain project files directly at its root.

```python
my_plugin.elyx
|-- refmap.yml       <- discovered
|-- metainfo.yml
`-- main.py
```

Do not add an extra wrapper directory:

```python
my_plugin.elyx
`-- my_plugin/
    |-- refmap.yml   <- not discovered at archive root
    `-- main.py
```

Directories declared explicitly in `refmap` must exist. For maximum compatibility with ZIP directory checks, include actual directory entries in the archive instead of storing only their child files.

## Installed layout

The engine extracts each plugin into its own directory under the app's plugin storage:

```python
<app files>/plugins/
|-- ElyxPlugins/
|   `-- <plugin_id>/
|       |-- refmap.yml
|       |-- main.py
|       `-- ...
|-- elyx_local_libs/
|   `-- <plugin_id>/
`-- archives/
```

These are runtime details, not import paths. Plugin code should use `elyx` environment values and normal local imports instead of constructing these paths.

## Recommended repository layout

For a project kept in Git, this layout scales well:

```python
my_plugin/
|-- .gitignore
|-- README.md
|-- refmap.yml
|-- plugin/
|   |-- meta.yml
|   |-- src/
|   |-- locales/
|   `-- res/
|-- wheels/
|-- builds/
`-- .elyx/
```

Ignore build output and caches:

```python
builds/
.elyx/cache/
**/__pycache__/
*.pyc
```

Do not ignore bundled wheels or assets if they are part of the released plugin.

[PreviousElyxBuilder](</docs/elyx/elyxbuilder>)[NextMetadata](</docs/elyx/metadata>)


---


# === DOCUMENT: Elyx Metadata (/docs/elyx/metadata) ===

---
title: "Elyx Metadata"
source: "https://plugins.exteragram.app/docs/elyx/metadata"
---

[Elyx](</docs/elyx>)

# Metadata

Declare identity, compatibility, dependencies, and localized descriptions.

# Plugin metadata

Structured Elyx plugins keep metadata outside the entry module. YAML is the recommended format:

```python
id: better_previews
name: Better Previews
description: "{plugin_description}"
author: "@yourUsername"
version: "1.2.0"
icon: exteraPlugins/1
app_version: ">=12.5.1"
sdk_version: ">=1.4.5.3"
requirements: "requests>=2.32, PyYAML>=6"
requires:
  helper_plugin (1.4.0): https://example.com/helper_plugin.elyx
```

## File discovery

Set an explicit path in `refmap.yml`:

```python
metainfo: plugin/meta.yml
```

or place one of these files at the archive root:

  * `metainfo.yaml`
  * `metainfo.yml`
  * `metainfo.json`

Keys surrounded by double underscores are also normalized when an explicit metadata file is used, so `__name__` becomes `name`. New projects should use plain names.

## Fields

Field| Required| Default| Meaning  
---|---|---|---  
`id`| yes| —| Stable plugin identifier  
`name`| yes| —| Display name  
`description`| no| `Description not provided`| Markdown-capable description  
`author`| no| `Unknown author`| Author name or Telegram handle  
`version`| no| `1.0.0`| Plugin release version  
`icon`| no| empty| Host icon reference, for example `exteraPlugins/1`  
`min_version`| no| empty| Legacy minimum app version  
`app_version`| no| empty| App compatibility expression  
`sdk_version`| no| empty| SDK compatibility expression  
`requirements`| no| empty| Packages installed through the plugin PIP controller  
`requires`| no| `{}`| Other plugins required at runtime  
  
### `id`

`id` must:

  * contain 2 to 32 characters
  * contain only ASCII letters, digits, and `_`
  * remain stable across updates

Valid:

```python
elyx_hello
weather2
MyPlugin
```

Invalid:

```python
a
my-plugin
my plugin
плагин
```

The identifier owns the plugin directory, settings, dependencies, enabled state, hooks, and menu items. Changing it creates a different plugin instead of updating the old one.

### `description`

Descriptions may contain Markdown and localization placeholders:

```python
description: "{plugin_description}\n\n[Source code](https://example.com)"
```

Given:

```python
# strings_en.yml
plugin_description: Adds better link previews.
```

Elyx resolves `{plugin_description}` when loading metadata. It first checks the current locale, then English. If a key is missing, its key name is used as the replacement text.

### Version fields

Use quoted YAML strings so parsers never reinterpret version values:

```python
version: "2.0.0-beta.1"
app_version: ">=12.5.1"
sdk_version: ">=1.4.5.3"
```

`app_version` and `sdk_version` are passed to the host plugin system. Declare the oldest versions on which you actually tested the APIs used by your plugin.

`min_version` is retained for legacy compatibility and performs a direct minimum-app-version check. Prefer `app_version` in new projects.

### `requirements`

This is either a comma-separated string:

```python
requirements: "requests>=2.32, tinydb>=4.8"
```

or an empty value when no packages are needed:

```python
requirements: ""
```

The engine parses the value and asks the host PIP controller to install the packages before importing `main`.

### `requires`

Declare dependencies on other plugins as a mapping:

```python
requires:
  helper_plugin: https://example.com/helper_plugin.elyx
  shared_runtime (2.1.0): https://example.com/shared_runtime.elyx
```

  * the mapping key is a plugin id
  * an optional minimum version goes in parentheses
  * the value is an optional download link shown by the installer
  * the required plugin must be installed and enabled

Required plugin ids follow the same 2–32 character rule.

## Access metadata at runtime

Use the plugin-bound `metainfo` value:

```bash
from elyx import metainfo
 
self.logger.info(f"Running {metainfo['name']} {metainfo['version']}")
```

`metainfo` is a dictionary containing normalized values and runtime-generated fields. Treat undocumented generated keys as read-only implementation details.

For optional environments, use:

```bash
from elyx import get_environment
 
environment = get_environment()
metadata = environment["metainfo"]
```

## Complete minimal metadata

```python
id: my_plugin
name: My Plugin
version: "1.0.0"
```

Everything else is optional, but production plugins should normally include `description`, `author`, `app_version`, and `sdk_version`.

[PreviousProject Structure](</docs/elyx/project-structure>)[NextModules and Imports](</docs/elyx/modules-and-imports>)


---


# === DOCUMENT: Elyx Modules and Imports (/docs/elyx/modules-and-imports) ===

---
title: "Elyx Modules and Imports"
source: "https://plugins.exteragram.app/docs/elyx/modules-and-imports"
---

[Elyx](</docs/elyx>)

# Modules and Imports

Split plugin code safely and use Elyx's isolated import system.

# Modules and imports

Every installed plugin receives its own internal module namespace:

```python
ElyxPlugins.<plugin_id>.*
```

You do not normally type this name. It prevents two plugins with identically named files from sharing entries in `sys.modules`.

## Import local modules

Given:

```python
main.py
helpers.py
features/
|-- __init__.py
`-- links.py
```

`main.py` can use familiar imports:

```python
from helpers import normalize_url
from features.links import build_preview
```

The same imports in another plugin resolve to that other plugin's files.

Relative imports also work:

```python
from .helpers import normalize_url
from .features.links import build_preview
```

For nested entry points, relative imports are often clearer:

```python
plugin/
|-- __init__.py
|-- main.py
`-- feature.py
```

```python
# plugin/main.py
from .feature import Feature
```

## Dynamic imports

Use the public Elyx helper when a module name is computed at runtime:

```bash
from elyx import import_module
 
module_name = "features.links"
links = import_module(module_name)
```

If a matching local module exists, `elyx.import_module` loads it from the calling plugin. Otherwise it falls back to Python's normal import system.

Do not rely on this for local modules:

```python
import importlib
 
importlib.import_module("features.links")
```

A plain dynamic import has no plugin context and may bypass Elyx's local resolution. Use `elyx.import_module`.

## Packages and namespace directories

Both regular packages and directories without `__init__.py` are supported:

```python
features/
|-- __init__.py     # optional
|-- links.py
`-- media/
    `-- images.py
```

Adding `__init__.py` is still recommended because it makes the same source tree work in desktop tooling and static analyzers.

## Supported Python files

Elyx can load:

  * `.py` source modules
  * `.pyc` bytecode modules
  * packages with `__init__.py` or `__init__.pyc`
  * namespace directories

Bytecode must match the Python runtime's magic number. This SDK uses Python 3.11, so compile release bytecode with Python 3.11.

> > [!NOTE] **Bytecode is version-specific** > A.pycbuilt with another Python version is rejected as incompatible. Bytecode is not a security boundary and does not protect secrets.

## Import data files as modules

JSON, YAML, YML, and TXT files can be imported when their stem is a valid module path.

Given `config.json`:

```json
{
  "api_url": "https://example.com",
  "limits": {
    "page_size": 50
  }
}
```

Use:

```python
import config
 
self.logger.info(config.api_url)
self.logger.info(str(config["limits"].page_size))
self.logger.info(str(config.get("missing", "fallback")))
```

For `notice.txt`:

```python
import notice
 
self.logger.info(notice.content)
```

Data modules are convenient for code-oriented configuration. Files which are part of the user-facing resource set usually belong under `assets` instead.

## Reserved and external top-level modules

The importer always leaves important SDK and Java roots to the normal runtime, including:

```python
android
androidx
base_plugin
client_utils
com
de
elyx
hook_utils
importlib
java
org
ui
```

Do not create local packages with those names. Also avoid shadowing Python standard-library names such as `json`, `pathlib`, or `typing`; local files with those names make code hard to reason about.

## Reload behavior

When Elyx reloads or unloads a plugin, it removes that plugin's namespaced modules and invalidates path caches. The next load executes fresh code.

This isolation covers Python modules. Process-global state outside the plugin namespace still needs explicit cleanup:

  * Java hooks and menu items are cleaned up by the host
  * threads, sockets, observers, and custom listeners created by your code must be stopped in `on_plugin_unload`
  * mutations to third-party modules remain global until the process restarts

```python
class MyPlugin(BasePlugin):
    def on_plugin_load(self):
        self.worker = start_worker()
 
    def on_plugin_unload(self):
        if self.worker:
            self.worker.stop()
            self.worker = None
```

## Import only the public Elyx facade

Supported:

```bash
from elyx import assets, settings, strings
from elyx import Asset, Callback, import_module
```

Engine, importer, installer, and plugin model modules are private. Do not discover or import them by physical file path; they can be renamed or reorganized without a deprecation period.

[PreviousMetadata](</docs/elyx/metadata>)[NextAssets](</docs/elyx/assets>)


---


# === DOCUMENT: Elyx Assets (/docs/elyx/assets) ===

---
title: "Elyx Assets"
source: "https://plugins.exteragram.app/docs/elyx/assets"
---

[Elyx](</docs/elyx>)

# Assets

Bundle and load images, SVG, Lottie, JSON, YAML, text, and binary resources.

# Assets

Declare a resource directory in `refmap.yml`:

```python
assets: plugin/res
```

Example tree:

```python
plugin/res/
|-- logo.svg
|-- loading.json
|-- banner.png
|-- defaults.yml
|-- about.txt
`-- icons/
    |-- add.svg
    `-- remove.svg
```

Then import the plugin-bound asset collection:

```bash
from elyx import assets
```

`assets` exists only when the plugin has a valid asset directory. If assets are optional, inspect the environment:

```bash
from elyx import get_environment
 
assets = get_environment().get("assets")
```

## Find an asset

There are three equivalent access styles:

```python
logo = assets.logo
logo = assets["logo"]
logo = assets.get("logo")
```

The extension may be included:

```toml
logo = assets["logo.svg"]
```

Nested paths are supported:

```toml
add_icon = assets["icons/add"]
add_icon = assets.icons.add
```

Attribute lookup uses a normalized filename stem. Characters other than ASCII letters and digits become `_`:

```python
empty-state.svg -> assets.empty_state
logo.dark.png   -> assets.logo
```

> > [!NOTE] **Avoid ambiguous stems** > Files such aslogo.svgandlogo.pnghave the same normalized stemlogo. Access them by full filename or give them distinct stems.

## `Assets`

An `Assets` object represents one directory.

```bash
from elyx import Assets
 
resource_count = len(assets)
has_logo = "logo" in assets
 
for filename in assets:
    self.logger.info(filename)
```

Important members:

Member| Result  
---|---  
`get(name)`| `Asset` for a file or `Assets` for a directory  
`obj[name]`| Alias for `get(name)`  
`obj.name`| Attribute-style lookup  
`parent`| `Assets` for the parent directory  
`dir_path`| Resource directory as `pathlib.Path`  
`len(obj)`| Number of direct children  
`name in obj`| Whether a direct child or normalized stem exists  
  
Missing items raise `AssetNotFoundException`.

## `Asset`

An `Asset` represents one file.

```python
logo = assets.logo
 
self.logger.info(logo.name)
self.logger.info(logo.filename)
self.logger.info(logo.ext)
self.logger.info(logo.path_str)
```

Path members:

Member| Type| Meaning  
---|---|---  
`name`| `str`| Normalized logical name  
`filename`| `str`| Actual filename  
`ext`| `str`| Last filename extension without `.`  
`path`| `pathlib.Path`| Absolute or resolved Python path  
`path_str`| `str`| String form of `path`  
`java_file`| `java.io.File`| Java file wrapper  
  
## Read content

```python
raw_bytes = assets.banner.content_bytes()
text = assets.about.content_string()
json_data = assets.loading.content_json()
yaml_data = assets.defaults.content_yaml()
automatic = assets.defaults.content()
```

`content()`:

  * parses `.json`, `.yaml`, and `.yml` into a dictionary
  * otherwise tries UTF-8 text
  * falls back to bytes when text decoding fails

Use the explicit method when the expected type matters.

## Android image helpers

### Regular drawables

```python
drawable = assets.banner.to_drawable()
location = assets.banner.to_image_location()
```

  * `to_drawable()` uses Android's `Drawable.createFromPath`
  * `to_image_location()` creates a Telegram `ImageLocation`

### Bitmap drawable

```python
bitmap_drawable = assets.banner.to_bitmap_drawable(width=64, height=64)
```

Width and height are bitmap pixels. Use this with bitmap-compatible files.

### SVG

```python
svg_drawable = assets.logo.to_svg_drawable()
svg_bitmap = assets.logo.to_svg_bitmap(width=48, height=48)
white_bitmap = assets.logo.to_svg_bitmap(48, 48, white=True)
```

Passing both dimensions to `to_svg_drawable` returns a scaled `BitmapDrawable`:

```python
scaled = assets.logo.to_svg_drawable(width=48, height=48)
```

Create a themed SVG thumbnail:

```python
from org.telegram.ui.ActionBar import Theme
 
thumb = assets.logo.to_svg_thumb(
    Theme.key_windowBackgroundWhiteBlueIcon,
    1.0,
)
```

### Lottie

```python
animation = assets.loading.to_lottie_drawable(width=32, height=32)
```

Lottie dimensions are converted through Android density helpers.

## Create an asset from another path

```bash
from elyx import Asset
 
asset = Asset.from_path("/absolute/path/to/file.svg")
```

`Asset.from_path` also accepts `pathlib.Path` and `java.io.File`.

For a temporary downloaded resource:

```bash
from elyx import Asset
 
remote = Asset.temp_asset_from_url(
    "https://example.com/icon.svg",
    "downloaded_icon.svg",
)
```

This is synchronous network and file I/O. Run it on a background queue, never directly in a UI callback or high-frequency hook.

## Errors

```bash
from elyx import AssetNotFoundException, AssetsDirNotFoundException
 
try:
    icon = assets["missing.svg"]
except AssetNotFoundException:
    icon = assets.logo
```

  * `AssetNotFoundException` means the requested file does not exist
  * `AssetsDirNotFoundException` means an asset root no longer exists
  * `ValueError` means a supplied path is not an allowed asset directory

Bundled asset paths are read-only from the plugin's point of view. Store mutable user data in the app data/cache helpers described in [File Utilities](</docs/file-utils>).

[PreviousModules and Imports](</docs/elyx/modules-and-imports>)[NextLocalization](</docs/elyx/localization>)


---


# === DOCUMENT: Elyx Localization (/docs/elyx/localization) ===

---
title: "Elyx Localization"
source: "https://plugins.exteragram.app/docs/elyx/localization"
---

[Elyx](</docs/elyx>)

# Localization

Keep translated strings outside Python and use locale fallback and formatting.

# Localization

Elyx loads translations from a file or directory declared in `refmap.yml`:

```python
strings: plugin/locales
```

Recommended layout:

```python
plugin/locales/
|-- strings_en.yml
|-- strings_ru.yml
`-- strings_de.json
```

## Locale filenames

The locale is the part after the final `_` in the filename stem:

```python
strings_en.yml   -> en
strings_ru.json  -> ru
ui_de.yaml       -> de
strings.yml      -> en
```

Supported formats:

  * `.yaml`
  * `.yml`
  * `.json`
  * `.py`

YAML or JSON is recommended. A Python localization file is executed while loading and only simple non-callable public values are collected.

## Translation files

`strings_en.yml`:

```python
plugin_description: Adds useful greeting actions.
title: Greetings
hello: "Hello, {name}!"
downloaded: "Downloaded {count} files"
file_forms:
  - file
  - files
  - files
```

`strings_ru.yml`:

```python
plugin_description: Добавляет полезные действия с приветствиями.
title: Приветствия
hello: "Привет, {name}!"
downloaded: "Загружено файлов: {count}"
file_forms:
  - файл
  - файла
  - файлов
```

Values may be strings, numbers, lists, or dictionaries.

## Read strings

Import the plugin-bound object:

```bash
from elyx import strings
```

All these forms are supported:

```toml
title = strings["title"]
title = strings.title
title = strings.get("title")
title = strings("title")
```

Calling the object also formats a string:

```python
message = strings("hello", name="Alice")
downloaded = strings("downloaded", count=5)
```

Positional placeholders work too:

```python
coordinates: "Position: {}, {}"
```

```python
text = strings("coordinates", 10, 20)
```

## Fallback order

For a regular lookup Elyx uses:

  1. the selected plugin locale
  2. English (`en`)
  3. the requested key itself

```python
label = strings("missing_key")
# "missing_key" when no locale defines it
```

Provide an explicit default:

```python
label = strings.get("optional_label", "Fallback label")
label = strings("optional_label", default="Fallback label")
```

Force a locale:

```python
russian_title = strings.get_with_locale("title", locale="ru")
english_title = strings("title", locale="en")
```

The selected locale comes from Elyx's language setting. When that setting is empty, the Android system locale is used.

## Plural forms

`Strings.pluralize` uses three Slavic-style forms:

```python
file_forms:
  - file
  - files
  - files
```

```python
text = strings.pluralize(21, "file_forms")
# "21 file"
```

For languages with different plural rules, select the form in plugin code or store complete formatted phrases. The current helper is not a full CLDR pluralization engine.

## Localize metadata

Place placeholders in `metainfo.yml`:

```python
description: "{plugin_description}"
```

Define the key in every locale:

```python
plugin_description: A localized plugin description.
```

Description placeholders are resolved when the plugin is loaded. They use the same current-locale-then-English fallback.

## Use `Strings` directly

The public class is useful for an independent in-memory catalog:

```bash
from elyx import Strings
 
labels = Strings({
    "en": {"save": "Save"},
    "ru": {"save": "Сохранить"},
})
 
button.setText(labels("save"))
```

Most plugins should use the environment-provided `strings`, because it already contains the files declared by the project.

## Best practices

  * always include English as the fallback locale
  * keep stable semantic keys such as `settings_privacy_title`
  * do not concatenate translated fragments to build a sentence
  * pass values through `{named}` placeholders
  * keep Markdown links and formatting consistent across translations
  * use YAML quotes around text containing `:`, `{}`, or leading special characters

If `from elyx import strings` fails, the localization path is missing, empty, or invalid. See [Troubleshooting](</docs/elyx/troubleshooting>).

[PreviousAssets](</docs/elyx/assets>)[NextSettings Storage](</docs/elyx/settings>)


---


# === DOCUMENT: Elyx Settings (/docs/elyx/settings) ===

---
title: "Elyx Settings"
source: "https://plugins.exteragram.app/docs/elyx/settings"
---

[Elyx](</docs/elyx>)

# Settings Storage

Read and write persistent plugin values through the Elyx environment.

# Settings storage

Every Elyx plugin receives a `SettingsController` bound to its metadata `id`:

```bash
from elyx import settings
```

This object stores values in the same host settings system used by `BasePlugin.get_setting` and settings rows from `ui.settings`.

## Read values

Equivalent forms:

```python
enabled = settings.get_setting("enabled", True)
enabled = settings.get("enabled", True)
enabled = settings("enabled", True)
enabled = settings["enabled"]
```

Bracket lookup has no explicit default and returns the host's default result when the key is absent. Prefer `get` when the fallback matters.

Read all plugin settings:

```python
values = settings.get_settings()
```

## Write values

```python
settings.set_setting("enabled", True)
settings.set("page_size", 50)
settings["username"] = "Alice"
```

To ask the host to rebuild an open settings screen:

```python
settings.set("mode", "advanced", reload_settings=True)
```

Use `reload_settings=True` only when the value changes which rows should be visible or how they should be rendered. A normal value update does not need a full reload.

## Clear values

```python
settings.clear_settings()
```

This removes all stored values for the current plugin id. It does not delete the plugin or reset files created elsewhere.

## Integrate with settings UI

The key used by a settings row is the same key visible through `settings`:

```bash
from typing import Any, List
 
from base_plugin import BasePlugin
from elyx import settings, strings
from ui.settings import Header, Input, Switch
 
 
class SettingsExamplePlugin(BasePlugin):
    def create_settings(self) -> List[Any]:
        return [
            Header(text=strings("settings_title")),
            Switch(
                key="enabled",
                text=strings("enabled"),
                default=True,
            ),
            Input(
                key="username",
                text=strings("username"),
                default="Alice",
            ),
        ]
 
    def on_plugin_load(self):
        if settings.get("enabled", True):
            self.logger.info(f"Hello {settings.get('username', 'Alice')}")
```

For all available UI rows and callbacks, see [Plugin Settings](</docs/plugin-settings>).

## `SettingsController`

You can create an explicit controller:

```bash
from elyx import SettingsController
 
my_settings = SettingsController("my_plugin")
```

This is mainly useful for shared library code which is given a plugin id. In a normal plugin, use the environment-provided `settings`; it cannot accidentally target the wrong id.

Method| Meaning  
---|---  
`get_settings()`| Return all stored values  
`get_setting(key, default=None)`| Read one value  
`get(key, default=None)`| Short alias  
`set_setting(key, value, reload_settings=False)`| Write one value  
`set(key, value, reload_settings=False)`| Short alias  
`clear_settings()`| Remove all values  
  
## Storage guidance

Settings are suitable for:

  * booleans, numbers, short strings, and small lists
  * UI preferences
  * lightweight persistent state

Use files or a database for:

  * large JSON objects
  * caches and downloaded media
  * frequently changing event history
  * secrets requiring dedicated secure storage

Settings survive plugin reloads and updates because they are keyed by plugin id. They are removed when the host clears or deletes that plugin's settings.

[PreviousLocalization](</docs/elyx/localization>)[NextDependencies](</docs/elyx/dependencies>)


---


# === DOCUMENT: Elyx Dependencies (/docs/elyx/dependencies) ===

---
title: "Elyx Dependencies"
source: "https://plugins.exteragram.app/docs/elyx/dependencies"
---

[Elyx](</docs/elyx>)

# Dependencies

Install remote Python packages, bundle local wheels, and depend on other plugins.

# Dependencies

Elyx supports three different dependency mechanisms:

  1. packages installed by the host PIP controller
  2. `.whl` files bundled inside the plugin
  3. other exteraGram plugins

Choose the smallest mechanism which fits the dependency.

## PIP requirements

Declare a comma-separated string in metadata:

```python
requirements: "requests>=2.32, tinydb>=4.8"
```

Before importing the entry module, Elyx asks the host PIP controller to install the packages for this plugin and adds their paths to `sys.path`.

Then use normal imports:

```python
import requests
from tinydb import TinyDB
```

See [PIP](</docs/pip>) and [Available Libraries](</docs/available-libraries>) for runtime constraints.

> > [!NOTE] **Android is not desktop Python** > Packages with native extensions need a build compatible with Python 3.11, Android, and the device ABI. A wheel built for Windows, macOS, or desktop Linux will not work on the phone.

## Bundled wheels

Point `refmap.yml` to a directory:

```python
wheels: wheels
```

```python
wheels/
|-- my_library-1.0.0-py3-none-any.whl
`-- another_library-2.1.0-py3-none-any.whl
```

On load, Elyx:

  1. finds direct `.whl` children of the directory
  2. extracts each wheel into plugin-specific local library storage
  3. skips a wheel already extracted under the same wheel filename stem
  4. removes stale extracted directories not present in the current release
  5. adds the installed wheel directories to Python's import path

Bundled wheels are useful when:

  * a pure-Python internal library is not published to an index
  * the exact artifact must travel with the plugin
  * installation must not depend on network access

Rebuild or rename the wheel when its contents change. Elyx uses the wheel stem as the extracted-version identity.

## Local source packages

Small libraries do not need a wheel. Put the package directly in the project:

```python
vendor/
|-- __init__.py
`-- parser.py
```

```python
from vendor.parser import parse
```

Local source is namespaced per plugin and participates naturally in live reload.

## Required plugins

Use the metadata `requires` mapping:

```python
requires:
  helper_plugin: https://example.com/helper_plugin.elyx
  shared_runtime (2.1.0): https://example.com/shared_runtime.elyx
```

The installer reports a missing, disabled, or outdated required plugin. A link can be offered for installation.

This declaration checks availability; it does not create a stable Python API between plugins. If plugins communicate, define an explicit compatibility contract and avoid importing another plugin's private namespaced modules.

## Version and update behavior

  * PIP requirements are associated with the plugin id
  * local wheels live in plugin-specific storage
  * obsolete bundled wheel directories are removed during wheel processing
  * deleting the plugin asks the host to uninstall its PIP dependencies
  * changing the plugin `id` creates a separate dependency scope

## Security

Treat dependencies as executable code:

  * pin or constrain important versions
  * distribute only wheels you built or verified
  * keep licenses and notices required by bundled libraries
  * do not place credentials in a wheel or source archive
  * test clean installation, update, disable/enable, and deletion

Elyx rejects path traversal while extracting bundled wheels, but that does not make an untrusted dependency safe to execute.

[PreviousSettings Storage](</docs/elyx/settings>)[NextDevelopment and Build](</docs/elyx/development>)


---


# === DOCUMENT: Elyx Development and Build (/docs/elyx/development) ===

---
title: "Elyx Development and Build"
source: "https://plugins.exteragram.app/docs/elyx/development"
---

[Elyx](</docs/elyx>)

# Development and Build

Develop on a PC, synchronize changes to a phone, build archives, and debug reloads.

# Development and build

The productive Elyx loop is:

  1. edit the complete project on a PC
  2. synchronize changed files to the installed plugin directory
  3. let Elyx unload and reload the plugin
  4. inspect errors and continue editing

## Prerequisites

  * a recent exteraGram build with plugin support
  * the current plugin SDK available to the IDE
  * Python 3.11 for bytecode-compatible builds
  * ADB when connecting a USB device or emulator
  * an Elyx archive installed once on the device

Enable the plugin engine and developer mode in exteraGram's plugin settings. The built-in development server listens on `127.0.0.1:42690` by default.

## Connect over ADB

Forward the phone's loopback server to the PC:

```python
adb forward tcp:42690 tcp:42690
```

Verify the device first if needed:

```python
adb devices
```

Only one process can use a local forwarded port. Remove and recreate the forward if another device was previously connected:

```python
adb forward --remove tcp:42690
adb forward tcp:42690 tcp:42690
```

## Initial installation

The structured live-reload protocol updates an Elyx plugin which is already known to the running engine. Before starting a watcher:

  1. build an `.elyx` or `.eaf` archive
  2. install it through exteraGram
  3. enable it at least once
  4. confirm that its `id` matches local `metainfo`

## Custom `elyx_dev_client.py`

Use the custom client from the project root. Client revisions may expose different flags, so inspect the script's own help:

```python
python elyx_dev_client.py --help
```

A client normally needs:

  * server host and port, usually `127.0.0.1:42690` after ADB forwarding
  * the local project root
  * the plugin `id`
  * ignore patterns for `.git`, `.elyx`, `builds`, `__pycache__`, and IDE files

The intended behavior is incremental: compare file hashes, send created, modified, moved, and deleted paths, then wait for the engine to reload the plugin.

> > [!NOTE] **The client is not bundled in this SDK repository** > Keep the client version used by your team alongside the plugin or in a shared development-tools repository. Its command-line interface is not part of the runtimeelyxAPI.

## Development protocol reference

The server exchanges consecutive JSON objects over a TCP connection. Elyx commands use:

  * `@` for the action name
  * `#` for the request id
  * `data` for base64-encoded, zlib-compressed JSON

Supported Elyx actions:

Action| Purpose  
---|---  
`elyx_ping`| Check that the Elyx server patch is active  
`get_elyx_plugins`| Return metadata for loaded Elyx plugins  
`elyx_compare_folder`| Compare client hashes with installed files  
`elyx_changes`| Apply file changes and reload a plugin  
  
Low-level tools should keep request ids unique and wait for the response with the matching `#`.

### Change records

An `elyx_changes` payload contains:

```json
{
  "plugin_id": "elyx_hello",
  "changes": [
    {
      "#": "modified",
      "path": "main.py",
      "is_directory": false,
      "content": "<base64 file bytes>"
    }
  ]
}
```

Event types are:

  * `created`
  * `modified`
  * `deleted`
  * `moved`

Created and modified files carry base64-encoded bytes. Moved entries use `path` and `dest_path`. Paths are relative to the plugin root.

After applying the batch, Elyx unloads the plugin, clears its imported modules, loads the project again, and reports `success` plus an `errors` list.

## Build with ElyxBuilder

ElyxBuilder can generate the entire initial project as well as package it:

```bash
python -m pip install --upgrade ElyxBuilder
elyb new
```

Run it from the directory containing `refmap.yml`:

```python
elyb build --ast -v -nf
```

For a compiled Python 3.11 build:

```python
elyb build -c 2 -v -nf
```

See [ElyxBuilder](</docs/elyx/elyxbuilder>) for `elyb new`, watch mode, incremental compilation, ignore lists, project statistics, and the official ElyxBuilder documentation.

> > [!NOTE] **Build with Python 3.11** > The phone runtime in this SDK is Python 3.11. Use Python 3.11 whenever the release contains.pycfiles.

## Build a plain archive

For source distributions, any ZIP tool is enough when it preserves paths. Archive the project contents so `refmap.yml` is at the ZIP root, then rename the result:

```python
my_plugin.zip -> my_plugin.elyx
```

Before publishing, inspect the archive:

```python
refmap.yml
metainfo.yml
main.py
src/...
assets/...
strings/...
```

Do not ship:

  * `.git/`
  * `.elyx/cache/`
  * `builds/`
  * `__pycache__/`
  * desktop virtual environments
  * secrets, API keys, signing material, or local logs

## Reload-safe code

Elyx clears local Python modules, hooks, and menu items. Your plugin remains responsible for resources it creates:

```python
class ReloadSafePlugin(BasePlugin):
    def on_plugin_load(self):
        self.closed = False
        self.worker = create_worker()
        self.worker.start()
 
    def on_plugin_unload(self):
        self.closed = True
        if self.worker:
            self.worker.stop()
            self.worker = None
```

Avoid starting permanent work at module import time. Put initialization in `on_plugin_load` and cleanup in `on_plugin_unload`.

## Release checklist

  * install from a clean archive, not only through live reload
  * test the default locale and English fallback
  * test with missing network access when dependencies are involved
  * disable and re-enable the plugin
  * update from the previous public version
  * verify every declared asset and localization path
  * inspect the archive root for accidental wrapper directories
  * confirm `id`, version constraints, and dependency links

[PreviousDependencies](</docs/elyx/dependencies>)[NextPublic API](</docs/elyx/public-api>)


---


# === DOCUMENT: Elyx Public API (/docs/elyx/public-api) ===

---
title: "Elyx Public API"
source: "https://plugins.exteragram.app/docs/elyx/public-api"
---

[Elyx](</docs/elyx>)

# Public API

Reference the stable symbols and plugin-bound values available from elyx.

# Public `elyx` API

Plugin code has one supported import surface:

```python
import elyx
```

or:

```bash
from elyx import Asset, SettingsController, strings
```

Do not import engine, importer, installer, or plugin model classes from their physical runtime modules. They are host implementation details.

## Plugin-bound environment

These names are resolved for the plugin which called them:

Name| Type| Availability  
---|---|---  
`settings`| `SettingsController`| always  
`metainfo`| `dict`| when metadata loaded successfully  
`refmap`| `dict`| when a refmap loaded successfully  
`assets`| `Assets`| when an asset directory is declared or discovered  
`strings`| `Strings`| when at least one localization value was loaded  
  

```bash
from elyx import assets, metainfo, settings, strings
```

These dynamic values are intentionally not included in `elyx.__all__`, because not every plugin declares every environment component.

### `get_environment()`

```bash
from elyx import get_environment
 
environment = get_environment()
settings = environment["settings"]
assets = environment.get("assets")
strings = environment.get("strings")
```

Returns the complete environment dictionary for the calling plugin. Raises `RuntimeError` when called outside Elyx plugin code.

### `import_module(name, package=None)`

```bash
from elyx import import_module
 
feature = import_module("features.preview")
```

Imports a matching module relative to the calling plugin. Falls back to normal Python import behavior when no local module exists.

## Assets

### `Assets(dir_path)`

Represents a resource directory. Supports `get`, item access, attribute access, nested directories, containment checks, iteration, and `len`.

See [Assets](</docs/elyx/assets>).

### `Asset(dir_path, filename, name)`

Represents one resource file.

Class methods:

```python
Asset.from_path(path)
Asset.temp_asset_from_url(url, filename)
```

Content methods:

```python
content_bytes()
content_string()
content_json()
content_yaml()
content()
```

Android conversion methods:

```python
to_image_location()
to_drawable()
to_bitmap_drawable(width=32, height=32)
to_lottie_drawable(width=32, height=32)
to_svg_drawable(width=None, height=None)
to_svg_thumb(color_key, alpha)
to_svg_bitmap(width=32, height=32, white=False)
```

### Asset exceptions

```bash
from elyx import AssetNotFoundException, AssetsDirNotFoundException
```

## Localization

### `Strings(all_strings)`

Locale-aware, read-only lookup object.

```python
catalog = Strings({
    "en": {"hello": "Hello, {name}!"},
    "ru": {"hello": "Привет, {name}!"},
})
 
catalog("hello", name="Alice")
catalog.get("hello")
catalog.get_with_locale("hello", locale="ru")
catalog.pluralize(5, "file_forms")
```

See [Localization](</docs/elyx/localization>).

## Settings

### `SettingsController(plugin_id)`

Persistent settings wrapper.

```python
controller.get_settings()
controller.get_setting(key, default=None)
controller.get(key, default=None)
controller.set_setting(key, value, reload_settings=False)
controller.set(key, value, reload_settings=False)
controller.clear_settings()
```

It also supports:

```toml
controller[key]
controller[key] = value
controller(key, default)
```

See [Settings storage](</docs/elyx/settings>).

## Java callback proxies

Elyx exports ready-to-instantiate proxy classes:

Name| Java interface| Implemented method| Returns callback value  
---|---|---|---  
`OnClickListener`| `android.view.View.OnClickListener`| `onClick`| no  
`Runnable`| `java.lang.Runnable`| `run`| no  
`Callback`| `Utilities.Callback`| `run`| no  
`Callback2`| `Utilities.Callback2`| `run`| no  
`Callback3`| `Utilities.Callback3`| `run`| no  
`CallbackReturn`| `Utilities.CallbackReturn`| `run`| yes  
  
Example:

```bash
from elyx import OnClickListener, Runnable
 
button.setOnClickListener(
    OnClickListener(lambda view: self.logger.info("Clicked"))
)
 
queue.postRunnable(
    Runnable(lambda: self.logger.info("Running"))
)
```

Extra constructor arguments are appended to the Java callback arguments:

```python
def on_click(view, item_id):
    self.logger.info(f"Clicked {item_id}")
 
button.setOnClickListener(OnClickListener(on_click, "save"))
```

Exceptions are caught and written to the app log so they do not cross the Java callback boundary.

## Proxy generators

### `gen(java_class, method_name, return_value=False, default_value=None)`

Creates and caches a Python proxy class for a one-method Java interface:

```bash
from android.content import DialogInterface
from elyx import gen
 
OnDismiss = gen(DialogInterface.OnDismissListener, "onDismiss")
dialog.setOnDismissListener(OnDismiss(lambda dialog: self.logger.info("Dismissed")))
```

For a callback which returns a value:

```python
Predicate = gen(
    SomeJavaPredicate,
    "test",
    return_value=True,
    default_value=False,
)
 
predicate = Predicate(lambda value: value is not None)
```

`default_value` is returned when the Python callback raises. It is valid only when `return_value=True`.

### `gen2(java_class, return_value=False, **methods)`

Creates a proxy for an interface with several methods:

```bash
from elyx import gen2
 
Listener = gen2(
    SomeJavaListener,
    onStart=lambda value: log_start(value),
    onFinish=lambda value: log_finish(value),
)
 
instance = Listener()
```

If the interface methods must return their Python results, set `return_value=True`.

## MVEL

### `mvel_execute(script, data, to_type=None, java_instance=None)`

Compiles and caches an MVEL expression, converts a Python dictionary to `java.util.HashMap`, and executes it:

```bash
from elyx import mvel_execute
 
visible = mvel_execute(
    "enabled && count > 0",
    {"enabled": True, "count": 3},
)
```

`data` must be a Python `dict` or Java `HashMap`.

Optional arguments:

  * `to_type` asks MVEL to convert the result to a Java type
  * `java_instance` provides the `this` value

## `LazyDict`

A dictionary with attribute lookup:

```bash
from elyx import LazyDict
 
value = LazyDict({"title": "Hello"})
self.logger.info(value.title)
```

Missing attributes behave like missing dictionary keys and raise `KeyError`.

## Supported exports

The current `elyx.__all__` is:

```python
(
    "Asset",
    "AssetNotFoundException",
    "Assets",
    "AssetsDirNotFoundException",
    "Callback",
    "Callback2",
    "Callback3",
    "CallbackReturn",
    "LazyDict",
    "OnClickListener",
    "Runnable",
    "SettingsController",
    "Strings",
    "gen",
    "gen2",
    "get_environment",
    "import_module",
    "mvel_execute",
)
```

The plugin-bound `assets`, `metainfo`, `refmap`, `settings`, and `strings` values are accessed explicitly and are not part of star imports.

[PreviousDevelopment and Build](</docs/elyx/development>)[NextTroubleshooting](</docs/elyx/troubleshooting>)


---


# === DOCUMENT: Elyx Troubleshooting (/docs/elyx/troubleshooting) ===

---
title: "Elyx Troubleshooting"
source: "https://plugins.exteragram.app/docs/elyx/troubleshooting"
---

[Elyx](</docs/elyx>)

# Troubleshooting

Diagnose installation, imports, resources, localization, dependencies, and live reload.

# Troubleshooting

Start with the error shown on the plugin page, then inspect exteraGram logs. Enable Elyx debug logs only while diagnosing a problem; verbose logging can be noisy.

## The archive is not recognized

Check:

  * the filename ends with `.elyx`, `.eaf`, `.elyx.zip`, or `.eaf.zip`
  * the file was downloaded completely
  * exteraGram's plugin engine and Elyx engine are available
  * the archive is a valid ZIP-compatible file

If it is an encrypted archive, open it through the installer and enter the correct password.

## “Metainfo not found”

The archive root must contain a discoverable metadata file or `refmap` must point to one:

```python
metainfo: plugin/meta.yml
```

Common mistake:

```python
archive/
`-- my_plugin/
    `-- refmap.yml
```

Correct:

```python
archive/
|-- refmap.yml
`-- plugin/
    `-- meta.yml
```

Also verify that YAML or JSON parses to a mapping and is not empty.

## Invalid plugin id

Use 2–32 ASCII letters, digits, and underscores:

```python
id: my_plugin_2
```

Hyphens, spaces, and non-ASCII letters are not accepted by the current Elyx validator.

## “Main file not found”

Check the path and capitalization:

```python
main: plugin/src/main.py
```

Paths are relative to the archive root and are case-sensitive on Android.

If `main` is omitted, only root-level `main.py` is discovered automatically.

## No `BasePlugin` subclass found

The entry module must expose a concrete subclass:

```python
from base_plugin import BasePlugin
 
 
class MyPlugin(BasePlugin):
    pass
```

Keep one plugin class in the entry module. Do not hide its definition behind a condition which is false during import.

## A local import fails

Given:

```python
src/
|-- __init__.py
`-- feature.py
```

Use:

```python
from src.feature import Feature
```

or, from another file inside `src`:

```python
from .feature import Feature
```

For a computed module name:

```bash
from elyx import import_module
 
feature = import_module("src.feature")
```

Other checks:

  * match filename capitalization exactly
  * add `__init__.py` for better desktop tooling
  * do not use reserved roots such as `elyx`, `ui`, `java`, or `android`
  * avoid files named after standard-library modules
  * rebuild Python 3.11 bytecode if `.pyc` is rejected

## `from elyx import assets` fails

`assets` is dynamic and exists only when Elyx has a valid asset root.

Check:

```python
assets: plugin/res
```

and verify that the directory exists in the archive.

For optional resources:

```bash
from elyx import get_environment
 
assets = get_environment().get("assets")
```

## An asset cannot be found

Inspect the exact filename:

```python
for filename in assets:
    self.logger.info(filename)
```

Try full filename access:

```toml
icon = assets["empty-state.svg"]
```

Remember that attribute names normalize punctuation to `_` and omit the extension.

## `from elyx import strings` fails

The strings environment is omitted when no translations were loaded.

Check:

  * the `strings` path in `refmap`
  * supported file extensions: JSON, YAML, YML, or Python
  * valid UTF-8 and valid YAML/JSON syntax
  * at least one public key/value pair

Always include an English catalog:

```python
strings/strings_en.yml
```

## A translation returns its key

That is the final fallback behavior. Add the key to the selected locale or English:

```python
missing_key: Visible text
```

If formatting fails, verify that placeholders match:

```python
hello: "Hello, {name}!"
```

```python
strings("hello", name="Alice")
```

## Settings do not update the screen

Writing a setting does not always rebuild an already open page:

```python
settings.set("mode", "advanced", reload_settings=True)
```

Use a normal write when only the value changes. Use `reload_settings=True` when row visibility or structure depends on it.

Make sure the settings row and code use the same key.

## A PIP dependency fails

Check:

  * package spelling and version constraints
  * whether the package is already provided by the SDK
  * Python 3.11 compatibility
  * Android and ABI support for native extensions
  * network availability during initial installation

Try a pure-Python version or bundle a verified compatible wheel. See [Dependencies](</docs/elyx/dependencies>).

## A bundled wheel does not update

Elyx identifies extracted wheels by filename stem. If wheel contents changed, build a wheel with a new version and filename:

```python
my_library-1.0.0-py3-none-any.whl
my_library-1.0.1-py3-none-any.whl
```

Do not overwrite the old wheel while keeping the same name.

## Live reload says the plugin was not found

The plugin must already be installed and loaded by Elyx. Verify:

  * local metadata `id` matches the installed id
  * the initial archive was installed
  * the engine completed startup
  * the plugin appears in `get_elyx_plugins`
  * ADB forwards local port `42690`

Reinstall the archive after changing `id`.

## Reload succeeds but old behavior remains

Elyx clears plugin-local modules, but process-global state can survive.

Look for:

  * background threads not stopped on unload
  * listeners or observers registered outside the host plugin APIs
  * monkey patches to shared modules
  * cached values stored in third-party modules
  * generated files read instead of the edited source

Move setup into `on_plugin_load` and cleanup into `on_plugin_unload`. As a final diagnostic, restart the app to distinguish reload state from source problems.

## The plugin works through live reload but not from an archive

This usually means the local working tree contains files excluded from the release or the archive has a different root.

Inspect the final archive and test a clean install. Confirm it contains:

```python
refmap.yml
the configured metadata file
the configured main file
all assets and locale files
all local modules and bundled wheels
```

## Reporting an Elyx problem

Include:

  * exteraGram version
  * plugin SDK version
  * Elyx version
  * device Android version and ABI
  * plugin metadata and `refmap`
  * a minimal archive or directory tree
  * the complete Python traceback
  * whether the problem occurs on clean install, enable, reload, or update

Remove tokens, personal data, private URLs, and credentials from logs before sharing them.

[PreviousPublic API](</docs/elyx/public-api>)[NextPlugin Class](</docs/plugin-class>)


---


# === DOCUMENT: Plugin Repositories (/docs/plugin-repositories) ===

---
title: "Plugin Repositories"
source: "https://plugins.exteragram.app/docs/plugin-repositories"
---

# Plugin Repositories

Publish plugins to the in-app catalog from a Telegram channel or a Git repository.

# Plugin repositories

The plugin catalog lists plugins from repositories. A repository is either a public Telegram channel or a Git repository with an index file. Users add repositories in the catalog, and the catalog shows the newest version of every plugin in each of them.

Catalog cards and update checks use the plugin metadata, so fill in at least `__id__`, `__name__`, `__version__`, and `__category__`. See [Plugin Class](</docs/plugin-class>) for `.plugin` files and [Metadata](</docs/elyx/metadata>) for Elyx archives.

## Telegram channels

Any public channel works as a repository. Post plugin files as documents.

  * The catalog reads the latest 2000 documents in the channel and keeps the files with a supported plugin extension.
  * Metadata is read from the files themselves. The app downloads them in the background, so `.plugin` files must be at most 2 MB and archives at most 10 MB. Larger files are not listed.
  * Posts that share a plugin `id` are versions of one plugin. The catalog shows the newest one, by post date or by the date of the last edit.
  * Editing a post and replacing its file publishes an update.

Share a channel repository with this link:

```python
https://t.me/exteraPluginRepo?c=<channel_username>
```

## Git repositories

A Git repository is read over plain HTTPS, without cloning and without a hosting API. Everything the catalog needs is in one index file, `extera-plugins.json`. You can use GitHub or any host that serves raw files.

### Layout

```python
my-plugins/
|-- extera-plugins.json
`-- plugins/
    |-- better_previews/
    |   `-- better_previews.plugin
    `-- weather/
        `-- weather.elyx
```

Only `extera-plugins.json` has a fixed name and place: it sits in the repository root, or in the directory the repository URL points to. Plugin files can live anywhere, because the index references each one by path.

### Repository URL

Users add a repository by pasting one of these URLs:

URL| Index location  
---|---  
`https://github.com/{owner}/{repo}`| `extera-plugins.json` on the default branch  
`https://github.com/{owner}/{repo}/tree/{ref}`| `extera-plugins.json` on branch or tag `{ref}`  
`https://github.com/{owner}/{repo}/tree/{ref}/{dir}`| `{dir}/extera-plugins.json` on `{ref}`  
`https://{host}/.../extera-plugins.json`| That file, on any host  
  
GitHub URLs are loaded from `raw.githubusercontent.com`. For GitLab, Codeberg, or your own server, give the direct raw URL of the index. Only `https` URLs are accepted. A `{ref}` must not contain `/`; for branches like `release/v2`, use the direct raw URL instead.

Share a Git repository with this link, where the parameter is the URL-encoded repository URL:

```python
https://t.me/exteraPluginRepo?git=https%3A%2F%2Fgithub.com%2Fyou%2Fmy-plugins
```

### Index file

```json
{
  "schema": 1,
  "name": "My Plugins",
  "description": "Small tweaks for chats and media.",
  "author": "@yourUsername",
  "homepage": "https://github.com/you/my-plugins",
  "plugins": [
    {
      "id": "better_previews",
      "name": "Better Previews",
      "description": "Modifies specific URLs for better previews",
      "author": "@yourUsername",
      "version": "1.2.0",
      "icon": "exteraPlugins/1",
      "app_version": ">=12.5.1",
      "sdk_version": ">=1.4.5.3",
      "requirements": "mpmath, tinydb",
      "category": ["chats", "tools"],
      "file": "plugins/better_previews/better_previews.plugin",
      "sha256": "9f2c4e0d1b7a6c3e8f5d2a1b0c9e8d7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d",
      "size": 18342,
      "updated_at": "2026-10-01T12:00:00Z"
    }
  ]
}
```

The file is UTF-8 JSON and must not exceed 1 MiB.

#### Repository fields

Field| Required| Meaning  
---|---|---  
`schema`| yes| Format version, currently `1`  
`name`| yes| Repository title shown in the catalog  
`description`| no| Short description of the repository  
`author`| no| Maintainer name or Telegram handle  
`homepage`| no| Page opened from the repository menu  
`plugins`| yes| List of plugin entries, may be empty  
  
#### Plugin fields

Field| Required| Meaning  
---|---|---  
`id`| yes| Plugin `id`, unique within the index  
`name`| yes| Display name  
`version`| yes| Version of the referenced file  
`file`| yes| Path to the plugin file, or an `https` URL  
`sha256`| yes| Lowercase hex SHA-256 of the file  
`size`| yes| File size in bytes  
`description`| no| Markdown-capable description  
`author`| no| Author name or Telegram handle  
`icon`| no| Icon reference, for example `exteraPlugins/1`  
`app_version`| no| App compatibility expression  
`sdk_version`| no| SDK compatibility expression  
`requirements`| no| PIP requirements as a comma-separated string or a list  
`category`| no| One category as a string, or up to three as a list  
`updated_at`| no| Time of the last file change, RFC 3339 in UTC  
  
Every field except `file`, `sha256`, `size`, and `updated_at` matches the metadata in the plugin file, with the same meaning and rules. For example, `category` uses the categories listed in [Plugin Class](</docs/plugin-class>). The index lists one entry per plugin, for its current version. It keeps no version history.

### Rules

  * **Relative paths.** A relative `file` is resolved against the URL the index was loaded from, so the repository works from any branch, tag, or mirror. Use absolute URLs only for files hosted elsewhere, such as GitHub Releases assets.
  * **Verification.** When the app downloads a file, its size and SHA-256 must match the index. The app then reads the metadata from the file, and `id` and `version` must match the entry. If any check fails, the plugin is not installed.
  * **Versions.** The app compares the leading numeric part of `version` component by component, so `1.10.0` is newer than `1.9.2` and suffixes like `-beta` are ignored. An entry with a newer version than the installed plugin is offered as an update.
  * **Compatibility.** Unknown fields are ignored, so newer tools can add fields without breaking older apps. If `schema` is higher than the app supports, the app does not load the repository and asks the user to update the app.
  * **Refreshing.** The app reloads the index at most once every 10 minutes, or when the user refreshes the catalog. It sends conditional requests, so an unchanged index costs one `304 Not Modified` response.

### Trust

Plugins from Git repositories install as plugins from an unknown source. Installing them requires **Unknown sources** in plugin settings, and the install sheet shows the security analysis result before the user confirms.

### Mirrors

When `raw.githubusercontent.com` is unreachable, the app may load a GitHub repository through jsDelivr (`cdn.jsdelivr.net/gh/{owner}/{repo}@{ref}`). jsDelivr caches branches, so it can lag behind recent pushes. It loads both the index and the files from the same mirror, so a lagging mirror shows an older version rather than a broken one. The fallback only works when the repository URL names a `{ref}`.

## Generating the index

Write the index with a script, not by hand: hashes and sizes go stale on every change. This script reads metadata from every plugin file under `plugins/` and writes `extera-plugins.json` next to itself. It needs [PyYAML](<https://pypi.org/project/PyYAML/>) only for Elyx archives with YAML metadata.

```python
#!/usr/bin/env python3
import ast
import hashlib
import json
import sys
import zipfile
from datetime import datetime, timezone
from pathlib import Path
 
ROOT = Path(__file__).resolve().parent
PLUGINS_DIR = ROOT / "plugins"
INDEX = ROOT / "extera-plugins.json"
ARCHIVE_SUFFIXES = {".elyx", ".eaf"}
 
REPOSITORY = {
    "schema": 1,
    "name": "My Plugins",
    "author": "@yourUsername",
    "homepage": "https://github.com/you/my-plugins",
}
 
FIELDS = (
    "id", "name", "description", "author", "version", "icon",
    "app_version", "sdk_version", "requirements", "category",
)
 
 
def read_plugin_file(path):
    meta = {}
    for node in ast.parse(path.read_text(encoding="utf-8")).body:
        if not isinstance(node, ast.Assign) or len(node.targets) != 1:
            continue
        target = node.targets[0]
        if isinstance(target, ast.Name) and target.id.startswith("__") and target.id.endswith("__"):
            key = target.id[2:-2]
            if key in FIELDS:
                meta[key] = ast.literal_eval(node.value)
    return meta
 
 
def load_structured(name, data):
    if name.endswith(".json"):
        return json.loads(data)
    import yaml
    return yaml.safe_load(data)
 
 
def read_archive(path):
    with zipfile.ZipFile(path) as archive:
        names = set(archive.namelist())
        meta_path = None
        for refmap in ("refmap.yaml", "refmap.yml", "refmap.json"):
            if refmap in names:
                meta_path = (load_structured(refmap, archive.read(refmap)) or {}).get("metainfo")
                break
        if meta_path is None:
            meta_path = next((n for n in ("metainfo.yaml", "metainfo.yml", "metainfo.json") if n in names), None)
        if meta_path is None:
            sys.exit(f"{path}: no metadata file")
        raw = load_structured(meta_path, archive.read(meta_path)) or {}
    return {key.strip("_"): value for key, value in raw.items() if key.strip("_") in FIELDS}
 
 
def main():
    previous = {}
    if INDEX.exists():
        previous = {p["id"]: p for p in json.loads(INDEX.read_text(encoding="utf-8")).get("plugins", [])}
 
    plugins = []
    for path in sorted(PLUGINS_DIR.rglob("*")):
        if not path.is_file():
            continue
        if path.suffix == ".plugin":
            meta = read_plugin_file(path)
        elif path.suffix in ARCHIVE_SUFFIXES:
            meta = read_archive(path)
        else:
            continue
 
        missing = [key for key in ("id", "name", "version") if not meta.get(key)]
        if missing:
            sys.exit(f"{path}: missing {', '.join(missing)}")
        if isinstance(meta.get("requirements"), list):
            meta["requirements"] = ", ".join(meta["requirements"])
 
        data = path.read_bytes()
        entry = {
            **meta,
            "file": path.relative_to(ROOT).as_posix(),
            "sha256": hashlib.sha256(data).hexdigest(),
            "size": len(data),
        }
        old = previous.get(meta["id"], {})
        if old.get("sha256") == entry["sha256"] and old.get("updated_at"):
            entry["updated_at"] = old["updated_at"]
        else:
            entry["updated_at"] = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
        plugins.append(entry)
 
    ids = [p["id"] for p in plugins]
    duplicates = sorted({i for i in ids if ids.count(i) > 1})
    if duplicates:
        sys.exit(f"duplicate plugin ids: {', '.join(duplicates)}")
 
    index = {**REPOSITORY, "plugins": plugins}
    INDEX.write_text(json.dumps(index, ensure_ascii=False, indent=2) + "\n", encoding="utf-8")
    print(f"{INDEX.name}: {len(plugins)} plugins")
 
 
if __name__ == "__main__":
    main()
```

To regenerate the index on every push, add a GitHub Actions workflow. Commits made with the workflow's own token do not trigger the workflow again.

```bash
name: Update plugin index
 
on:
  push:
    branches: [main]
    paths: ["plugins/**", "build_index.py"]
 
permissions:
  contents: write
 
jobs:
  index:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install pyyaml
      - run: python build_index.py
      - run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add extera-plugins.json
          git diff --cached --quiet || git commit -m "chore: update plugin index"
          git push
```

## Checklist

  * Every plugin has a unique, stable `id` that never changes between versions.
  * `version` goes up with every release you want offered as an update.
  * `category` is set, so the plugin shows up under catalog filters.
  * For Git repositories, `extera-plugins.json` is regenerated after every change to a plugin file, and its `file` paths are relative.

[PreviousBulletin Helper](</docs/bulletin-helper>)[NextDevelopment Server](</docs/dev-server>)


---


# === DOCUMENT: Development Server (/docs/dev-server) ===

---
title: "Development Server"
source: "https://plugins.exteragram.app/docs/dev-server"
---

# Development Server

Connect desktop tooling to exteraGram for plugin updates, reloads, and remote debugging.

# Development server

The SDK includes a small TCP development server for desktop tooling. It can inspect and reload regular Python plugins, and Elyx extends it with multi-file synchronization.

The default endpoint is:

```python
127.0.0.1:42690
```

Because the server binds to the phone's loopback interface, connect through ADB:

```python
adb devices
adb forward tcp:42690 tcp:42690
```

Then desktop clients connect to `127.0.0.1:42690`.

> > [!NOTE] **Development use only** > Enable the server only while developing. Its protocol can install, rewrite, enable, disable, reload, and delete plugin code.

## Message format

The connection carries consecutive UTF-8 JSON objects. A basic request uses:

```json
{
  "@": "ping",
  "#": "request-1"
}
```

  * `@` is the command name
  * `#` is a client-generated request id
  * additional fields depend on the command

The response repeats the request id:

```json
{
  "#": "request-1",
  "pong": true
}
```

Clients should keep request ids unique while requests are in flight and match responses by `#`.

## Core commands

Command| Additional fields| Purpose  
---|---|---  
`ping`| —| Test the connection  
`get_plugins`| —| List loaded plugins and their current state  
`enable_plugin`| `plugin_id`| Enable an installed plugin  
`disable_plugin`| `plugin_id`| Disable an installed plugin  
`reload_plugin`| `plugin_id`| Reload a regular single-file plugin  
`write_plugin`| `plugin_id`, `content`| Install or update Python source  
`remove_plugin`| `plugin_id`| Delete an installed plugin  
`start_debugger`| `host`, `port`, `platform`| Start a remote debugger  
`stop_debugger`| `platform`| Stop a remote debugger  
  
`write_plugin.content` contains the complete Python source as a JSON string. The server writes it to a temporary file and passes it through the normal plugin loader.

## Elyx multi-file commands

Elyx adds incremental folder comparison and change batches:

Command| Purpose  
---|---  
`elyx_ping`| Check that Elyx development support is active  
`get_elyx_plugins`| List loaded structured plugins  
`elyx_compare_folder`| Compare local hashes with installed files  
`elyx_changes`| Apply file changes and reload one Elyx plugin  
  
Elyx payloads are compressed JSON carried in a base64 `data` field. See [Elyx development and build](</docs/elyx/development>) for the payload format, initial installation requirement, and `elyx_dev_client.py` workflow.

## Remote debugging

The server accepts:

```json
{
  "@": "start_debugger",
  "#": "debug-1",
  "host": "127.0.0.1",
  "port": 5678,
  "platform": "vscode"
}
```

Supported platform values:

  * `vscode`
  * `pycharm`

The debugger port is separate from `42690`. Configure the IDE and any required ADB reverse/forward rule for the direction used by that debugger.

VS Code attach example:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach to exteraGram",
      "type": "debugpy",
      "request": "attach",
      "connect": {
        "host": "127.0.0.1",
        "port": 5678
      },
      "pathMappings": [
        {
          "localRoot": "${workspaceFolder}",
          "remoteRoot": "/data/user/0/com.exteragram.messenger/files/plugins"
        }
      ]
    }
  ]
}
```

Debugger support also depends on the corresponding Python debugger package being available in the app runtime.

## Client behavior

A robust custom client should:

  * test `ping` or `elyx_ping` before sending changes
  * apply socket timeouts
  * preserve request ids
  * display complete server errors
  * ignore build caches and version-control directories
  * debounce rapid editor events into one change batch
  * reconnect after the app restarts

The desktop client's CLI is not part of the SDK runtime API. Keep the exact client version and its usage instructions with the plugin project.

## Connection troubleshooting

If the client cannot connect:

  1. confirm developer mode and the server are enabled
  2. run `adb devices`
  3. recreate `adb forward tcp:42690 tcp:42690`
  4. check that no other process owns local port `42690`
  5. restart the server after changing device

If core `ping` works but `elyx_ping` does not, the regular server is running but Elyx development support has not initialized.

[PreviousPlugin Repositories](</docs/plugin-repositories>)[NextPIP & Dependencies](</docs/pip>)


---


# === DOCUMENT: PIP & Dependencies (/docs/pip) ===

---
title: "PIP & Dependencies"
source: "https://plugins.exteragram.app/docs/pip"
---

# PIP & Dependencies

Learn how to use external Python libraries in your plugins using the built-in PIP-like package manager.

The plugin system includes a built-in package manager that can automatically download and install dependencies from [PyPI](<https://pypi.org/>).

## Declaring Dependencies

To use external libraries, you need to list them in your plugin's metadata using the `__requirements__` variable. This should be a list of strings following the standard PEP 508 requirement format.

```python
__id__ = "inline_calculator"
__name__ = "Inline Calculator"
__requirements__ = ["mpmath", "tinydb"]
 
from base_plugin import BasePlugin, HookResult, HookStrategy
from mpmath import mp
from tinydb import TinyDB
 
class CalculatorPlugin(BasePlugin):
    ...
```

When a user installs your plugin, exteraGram will automatically download these libraries and their dependencies.

## How it Works

  1. **Resolution** : The system fetches package metadata from PyPI to find a version that satisfies your constraints.
  2. **Pure-Python Only** : It only downloads "pure-Python" wheels (files ending in `-none-any.whl`).
  3. **Isolation** : Libraries are installed in a shared `libs` directory. If multiple plugins require the same version of a library, it is only downloaded once.
  4. **Automatic Cleanup** : When you uninstall a plugin, any dependencies that are no longer needed by any other plugin are automatically removed.

## Limitations

The built-in package manager has several important limitations compared to a full `pip` installation:

### 1\. No Binary Extensions

**The most significant limitation.** Many popular Python libraries (like `numpy`, `pandas`, `scipy`, `cryptography`, or `opencv`) contain C/C++ extensions. These **cannot** be installed because they must be compiled for the specific Android architecture and linked against the application's Python runtime.

  * **Allowed** : Packages that say "Universal" or "none-any" in their wheel filename.
  * **Forbidden** : Packages that require a specific platform (e.g., `manylinux`, `macosx`, `win_amd64`) or have compiled components.

### 2\. Dependency Conflicts

Unlike a virtual environment, all plugins share the same set of installed libraries.

  * If **Plugin A** requires `requests==2.25.1` and **Plugin B** requires `requests==2.31.0`, a conflict occurs.
  * The system will attempt to use a version that satisfies both. If no such version is currently active or available, the installation of the second plugin will fail.

### 3\. Network Access

Installation requires an active internet connection to reach `pypi.org` and download files.

### 4\. Limited Marker Support

The system supports basic environment markers (like `python_version`, `sys_platform`), but complex markers or those referring to `extra` might not work as expected.

## Pre-installed Libraries

Some libraries are pre-installed in the application for performance and stability. You don't need to list them in `__requirements__`, but doing so is harmless.

Check the [Available Libraries](</docs/available-libraries>) page for the full list.

## Troubleshooting

If a plugin fails to install due to a dependency error:

  * **"No pure-Python wheel found"** : The library you are trying to use contains compiled code. You cannot use this library unless it is manually added to the core application.
  * **"Dependency conflict"** : Another installed plugin is using an incompatible version of the same library.
  * **"Package not found"** : Check the spelling of the package name or its availability on PyPI.

[PreviousDevelopment Server](</docs/dev-server>)[NextAvailable Libraries](</docs/available-libraries>)


---


# === DOCUMENT: Available Libraries (/docs/available-libraries) ===

---
title: "Available Libraries"
source: "https://plugins.exteragram.app/docs/available-libraries"
---

# Available Libraries

A list of pre-installed Python libraries available in the plugin environment.

The plugin environment comes with a specific version of Python and a set of pre-installed third-party libraries that you can use in your plugins without any extra setup.

## Python Version

  * **Python** : `3.11`

## Pre-installed Pip Packages

You can directly import and use the following libraries in your plugin code:

  * **`beautifulsoup4`** : A library for pulling data out of HTML and XML files. Useful for web scraping.
  * **`debugpy`** : The official debugger for Python from Microsoft, enabling remote debugging capabilities (used by the Dev Server).
  * **`lxml`** : A powerful and Pythonic library for processing XML and HTML.
  * **`packaging`** : Core utilities for Python packages.
  * **`pillow`** : The friendly PIL fork (Python Imaging Library). Useful for image manipulation.
  * **`requests`** : A simple, yet elegant, HTTP library. Essential for making web requests.
  * **`PyYAML`** : A YAML parser and emitter for Python.

## Using Other Libraries

If your plugin requires a library that is **not** on this list, you can use the built-in [PIP-like package manager](</docs/pip>) by declaring it in your plugin's `__requirements__`.

However, please note that **binary extensions are not supported**. Only pure-Python libraries can be installed. For more details, see the [PIP & Dependencies](</docs/pip>) page.

[PreviousPIP & Dependencies](</docs/pip>)[NextCommon Telegram Classes](</docs/common-source-classes>)


---


# === DOCUMENT: Common Telegram Classes (/docs/common-source-classes) ===

---
title: "Common Telegram Classes"
source: "https://plugins.exteragram.app/docs/common-source-classes"
---

# Common Telegram Classes

Reference for frequently used Telegram source classes and their locations for plugin development.

## Links to commonly used Telegram classes

It is recommended to have a local copy of Telegram sources, opened in Android Studio.

### LaunchActivity

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java`

That's where app initialization happens. Custom links are handled here as well.

### ProfileActivity

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/ProfileActivity.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/ProfileActivity.java`

User and channel profile fragment.

### ChatActivity

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java`

Chat rendering and functionality.

### MessageObject

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/messenger/MessageObject.java>)

Path: `TMessagesProj/src/main/java/org/telegram/messenger/MessageObject.java`

Wrapper for `TLRPC.Message`.

### ChatMessageCell

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/Cells/ChatMessageCell.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/Cells/ChatMessageCell.java`

Handles message rendering.

### AndroidUtilities

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/messenger/AndroidUtilities.java>)

Path: `TMessagesProj/src/main/java/org/telegram/messenger/AndroidUtilities.java`

Contains a lot of useful utilities.

### MessagesController

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java>)

Path: `TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java`

Contains all methods related to managing app state and Telegram requests.

### MessagesStorage

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java>)

Path: `TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java`

Manages local database state. You may use `database` field to execute custom SQLite queries.

### SendMessagesHelper

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java>)

Path: `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java`

Contains methods for sending all kind of messages, including files.

### BulletinFactory

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/Components/BulletinFactory.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/Components/BulletinFactory.java`

Bulletins are small notification messages displayed on the bottom.

### AlertDialog

[View on GitHub](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/ui/ActionBar/AlertDialog.java>)

Path: `TMessagesProj/src/main/java/org/telegram/ui/ActionBar/AlertDialog.java`

Alert dialogs are shown on top of current fragment. They support custom handlers on action buttons.

### TLRPC (all Telegram request models)

Path: `TMessagesProj/src/main/java/org/telegram/tgnet`

  * [TLRPC.java](<https://github.com/DrKLO/Telegram/blob/master/TMessagesProj/src/main/java/org/telegram/tgnet/TLRPC.java>)
  * [Additional models](<https://github.com/DrKLO/Telegram/tree/master/TMessagesProj/src/main/java/org/telegram/tgnet/tl>)

Human-readable list: <https://corefork.telegram.org/schema> (not always up-to-date)

[PreviousAvailable Libraries](</docs/available-libraries>)


---
