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
