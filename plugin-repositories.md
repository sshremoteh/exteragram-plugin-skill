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
