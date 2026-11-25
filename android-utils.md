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
