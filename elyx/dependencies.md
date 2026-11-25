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
