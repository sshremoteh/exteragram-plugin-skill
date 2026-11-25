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
