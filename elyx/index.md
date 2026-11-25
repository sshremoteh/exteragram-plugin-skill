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
