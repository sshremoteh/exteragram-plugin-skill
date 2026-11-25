---
name: exteragram-plugin-developer
description: Expert guidance for developing, testing, and debugging Python plugins for the exteraGram Android client (SDK v1.4.4.3+, app baseline 12.5.1+, Python 3.11, Chaquopy v16). Activate this skill whenever the user mentions exteraGram, developing plugins for exteraGram, BasePlugin, Telegram Android mods/hooks, Elyx framework, Xposed method hooking in exteraGram, or wants to write, fix, or build an exteraGram plugin.
---

# exteraGram Plugin Developer Skill

This skill provides expert rules, patterns, code templates, and constraints for writing robust plugins for the **exteraGram** Telegram Android client.

---

## 1. Environment & Architecture Constraints

- **Language & Runtime**: Python 3.11 via **Chaquopy v16** running inside the Android exteraGram process.
- **Baseline Requirements**:
  - exteraGram version: `>=12.5.1` (older versions trigger safe mode and refuse to load plugins).
  - SDK version: `>=1.4.4.3`.
- **Complete Reference Docs**: When deep documentation or specific API details are needed, consult the Markdown documentation bundled in this skill directory:
  - Base docs: `./` (e.g., `plugin-class.md`, `plugin-settings.md`, `xposed-hooking.md`, `client-utils.md`)
  - Elyx framework: `./elyx/`
  - Single-file consolidated manual: `./FULL_AI_GUIDE.md`
  - Quick cheatsheet: `./AI_CHEATSHEET.md`

---

## 2. Core Plugin Requirements & Constraints

### A. Static Metadata (Strict Rule)
Plugin metadata is extracted by exteraGram statically using **Python AST**.
- **MUST**: Be plain string constants defined at the top level of the plugin file.
- **NEVER**: Compute metadata dynamically (no functions, f-strings, conditional expressions, or imported variables).

```python
__id__ = "my_plugin_unique_id"      # Unique snake_case identifier
__name__ = "Plugin Display Name"
__description__ = "Short description of what the plugin does"
__author__ = "Author"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"         # Vector icon or asset path
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"
```

### B. Hook Registration (Strict Rule)
Defining a hook method (e.g. `on_send_message_hook`) does **NOT** register it automatically!
- You **MUST** explicitly register each hook inside `on_plugin_load(self)` using the corresponding registration helper:
  - Outgoing messages: `self.add_on_send_message_hook()`
  - Incoming updates: `self.add_on_update_hook()`
  - Requests: `self.add_on_request_hook()`
  - Responses: `self.add_on_response_hook()`
  - Menu items: `self.add_chat_menu_item(...)`, `self.add_message_menu_item(...)`

### C. HookResult Return Values
Every event hook must return a `HookResult` object:
```python
from base_plugin import HookResult, HookStrategy

# 1. Do nothing / keep original behaviour:
return HookResult()

# 2. Modify parameters:
params.message = "Modified text"
return HookResult(strategy=HookStrategy.MODIFY, params=params)

# 3. Intercept and completely cancel the event:
return HookResult(strategy=HookStrategy.CANCEL)
```

### D. Threading & UI Safety
- **Android UI Thread**: Hooks and settings UI execute directly on the Android main thread.
- **NEVER** perform network requests, DNS queries, blocking socket calls, or slow file operations directly in a hook or on the UI thread.
- Use `run_on_queue(...)` from `client_utils` to run long-running background tasks.
- When calling UI methods (toasts, dialogs, bulletins) from a background thread or queue, wrap them in `run_on_ui_thread(...)` from `android_utils`.

---

## 3. Standard Single-File Plugin Template

Use this boilerplate as the default standard for single-file plugins:

```python
from typing import Any, List
from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Header, Input, Switch, Selector, Text
from client_utils import run_on_queue
from android_utils import run_on_ui_thread, show_toast
from ui.bulletin import BulletinHelper
# BulletinHelper.show_info("text"), BulletinHelper.show_success("text"), BulletinHelper.show_error("text")

__id__ = "echo_helper"
__name__ = "Echo Helper"
__description__ = "Demonstrates command processing and settings in exteraGram"
__author__ = "Developer"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"

DEFAULT_PREFIX = "."

class EchoHelperPlugin(BasePlugin):
    def on_plugin_load(self) -> None:
        # Mandatory: explicitly register hooks
        self.add_on_send_message_hook()
        self.logger.info("EchoHelperPlugin loaded successfully")

    def on_plugin_unload(self) -> None:
        self.logger.info("EchoHelperPlugin unloaded")

    def create_settings(self) -> List[Any]:
        return [
            Header(text="Echo Configuration"),
            Input(
                key="command_prefix",
                text="Command Prefix",
                default=DEFAULT_PREFIX,
                subtext="Character to trigger commands (default: .)",
                icon="msg_edit",
            ),
            Switch(
                key="enable_bulletin",
                text="Show Bulletin Notification",
                default=True,
                icon="msg_info",
            ),
            Text(
                text="About",
                subtext="Created for exteraGram Python SDK",
                icon="msg_info",
            ),
        ]

    def on_send_message_hook(self, account: int, params: Any) -> HookResult:
        # Safety check: media sends or sticker sends might have params.message = None
        if not hasattr(params, "message") or not isinstance(params.message, str):
            return HookResult()

        text = params.message.strip()
        prefix = self.get_setting("command_prefix", DEFAULT_PREFIX)

        if text.startswith(f"{prefix}echo "):
            user_input = text[len(f"{prefix}echo "):].strip()
            params.message = f"Echo: {user_input}"

            if self.get_setting("enable_bulletin", True):
                try:
                    BulletinHelper.show_success("Echo: message rewritten")
                except Exception:
                    pass

            return HookResult(strategy=HookStrategy.MODIFY, params=params)

        return HookResult()
```

---

## 4. Key SDK Modules & APIs

### `client_utils`
- `send_message(account, peer, message, reply_to_msg_id=0)`
- `edit_message(account, peer, message_id, new_text)`
- `run_on_queue(queue_name, callable_func, *args, **kwargs)` — run tasks asynchronously off the main UI thread.
- `get_account_instance(account)` — returns Java `AccountInstance` for the given account ID.

### `android_utils`
- `run_on_ui_thread(callable_func, *args, **kwargs)`
- `show_toast(text, is_long=False)`
- `copy_to_clipboard(text)`

### `ui.settings`
Components for constructing `create_settings(self)`:
- `Header(text="...")` — section title.
- `Input(key="key", text="...", default="...", subtext="...", icon="...")` — text input field.
- `Switch(key="key", text="...", default=True, icon="...")` — toggle switch.
- `Selector(key="key", text="...", items=[...], default=0, icon="...")` — dropdown / dialog picker.
- `Text(text="...", subtext="...", icon="...")` — read-only informative row.
- `Button(text="...", on_click=callable, icon="...")` — clickable action button.

### `ui.alert` & `ui.bulletin`
- `from ui.alert import AlertDialogBuilder`:
  `builder = AlertDialogBuilder(fragment)` — builds standard Telegram modal popups with buttons and text.
- `from ui.bulletin import BulletinHelper`:
  `BulletinHelper.show_info("text")` — informational floating bulletin.
  `BulletinHelper.show_success("text")` — success bulletin with check icon.
  `BulletinHelper.show_error("text")` — error bulletin.

### Java Interop & Hooking (`hook_utils` & `xposed_hooking`)
- `find_class("org.telegram.messenger.MessagesController")`
- `hook_method(class_ref, "methodName", hook_callback, *param_types)`
- Callback receives `param`:
  - `param.args` — list of Java arguments.
  - `param.thisObject` — Java instance (`this`).
  - `param.setResult(value)` — override return value.

---

## 5. Elyx Framework (Modular Plugins)

For multi-file projects:
- Layout:
  ```
  my-plugin/
  ├── elyx.toml             # Configuration & package metadata
  ├── src/
  │   ├── main.py           # Entry point (inherits BasePlugin)
  │   └── modules/          # Submodules
  ├── assets/               # Icons, images
  └── locales/              # en.json, ru.json
  ```
- CLI:
  - `elyx build` — packages project into a distribution `.zip` / `.extera` plugin package.
  - `elyx run` — sends plugin to a connected development device / server.

---

## 6. Development Workflow Checklist

When asked to write or inspect an exteraGram plugin:
1. **Verify metadata**: Ensure `__id__`, `__name__`, `__version__`, `__app_version__ = ">=12.5.1"`, `__sdk_version__ = ">=1.4.4.3"` are top-level constants.
2. **Verify hook registrations**: Confirm `self.add_on_*_hook()` calls are present in `on_plugin_load()`.
3. **Verify message safety**: Ensure `params.message` is checked for `None` or `str` before accessing string methods.
4. **Verify thread safety**: Ensure all HTTP/IO tasks are dispatched via `run_on_queue`.
5. **Verify settings**: Default values should be provided on all `self.get_setting("key", default)`.
