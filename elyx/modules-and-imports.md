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
