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
