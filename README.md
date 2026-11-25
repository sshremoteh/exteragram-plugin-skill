# exteraGram Plugin Developer — AI Agent Skill & Docs

Полная текстовая документация и готовый **AI Agent Skill** для разработки плагинов для **exteraGram** (Python SDK v1.4.4.3+, app baseline: exteraGram 12.5.1+, Python 3.11, Chaquopy v16).

Репозиторий оптимизирован для использования с любыми ИИ-агентами (Claude Code, Antigravity, Cursor, Windsurf, Codex, OpenCode и др.) — просто подключите его как скилл и попросите агента написать плагин.

---

## Установка скилла для ИИ-агентов

### Через `skills` CLI (универсально для большинства агентов)
```bash
npx skills add sshremoteh/exteragram-plugin-skill
```

### Вручную через `git clone`

**В текущий проект** (подхватывается Antigravity, Codex, OpenCode, Cursor и др.):
```bash
git clone https://github.com/sshremoteh/exteragram-plugin-skill.git .agents/skills/exteragram-plugin-developer
```

**Глобально для Claude Code:**
```bash
git clone https://github.com/sshremoteh/exteragram-plugin-skill.git ~/.claude/skills/exteragram-plugin-developer
```

**Глобально для других агентов (`.agents/skills`):**
```bash
git clone https://github.com/sshremoteh/exteragram-plugin-skill.git ~/.agents/skills/exteragram-plugin-developer
```

**Для веб-чатов (ChatGPT, Claude.ai и т.д.):**
Загрузите файл [FULL_AI_GUIDE.md](FULL_AI_GUIDE.md) (полная база знаний в одном файле) или [AI_CHEATSHEET.md](AI_CHEATSHEET.md) (компактная шпаргалка) в контекст диалога.

---

## Быстрая навигация по разделам

### Getting Started
- [Introduction](introduction.md) — General introduction to exteraGram plugin SDK
- [Setup](setup.md) — SDK installation, environment setup, and prerequisites
- [First Plugin](first-plugin.md) — Walkthrough creating your first exteraGram plugin

### Core Plugin Architecture
- [Plugin Class](plugin-class.md) — BasePlugin lifecycle, hooks, and core methods
- [Multi-Account](multi-account.md) — Handling multi-account Telegram environment
- [Plugin Settings](plugin-settings.md) — Settings UI components (Header, Input, Switch, etc.)
- [Logging](logging.md) — Plugin logging utilities

### Java Interop & Method Hooking
- [Class Proxy](class-proxy.md) — Creating dynamic Java proxy implementations from Python
- [Xposed Method Hooking](xposed-hooking.md) — Method hooking, constructor hooking via Xposed helpers
- [Hook Utilities (Reflection)](hook-utils.md) — Java reflection helpers, find_class, call_method, etc.

### Utilities & UI Components
- [Android Utilities](android-utils.md) — UI thread execution, listeners, clipboard, toasts
- [Client Utilities](client-utils.md) — Sending/editing messages, queues, controllers, requests
- [Text Formatting](text-formatting.md) — Telegram markdown and entity formatting
- [File Utilities](file-utils.md) — File system and path helpers for plugins
- [Intents](intents.md) — Android Intents helper
- [Alert Dialog Builder](alert-dialog-builder.md) — Creating native Telegram alert dialogs
- [Bulletin Helper](bulletin-helper.md) — Showing Telegram floating bulletins/banners

### Elyx Framework (Modular Plugins)
- [Elyx Overview](elyx/index.md) — Modern framework for modular exteraGram plugins
- [Elyx Quick Start](elyx/quick-start.md) — Quick start guide for Elyx
- [ElyxBuilder CLI](elyx/elyxbuilder.md) — CLI tool to scaffold and build Elyx plugins
- [Elyx Project Structure](elyx/project-structure.md) — Standard folder layout and structure
- [Elyx Metadata](elyx/metadata.md) — Plugin metadata declaration in Elyx
- [Elyx Modules and Imports](elyx/modules-and-imports.md) — Managing modules and local imports
- [Elyx Assets](elyx/assets.md) — Managing icons, images, and other assets
- [Elyx Localization](elyx/localization.md) — Multi-language string translations
- [Elyx Settings](elyx/settings.md) — Simplified type-safe settings storage
- [Elyx Dependencies](elyx/dependencies.md) — Packaging external python wheels and libraries
- [Elyx Development and Build](elyx/development.md) — Development workflow, hot reloading, building
- [Elyx Public API](elyx/public-api.md) — Public API reference for Elyx
- [Elyx Troubleshooting](elyx/troubleshooting.md) — Common errors and debugging

### Tooling & Ecosystem
- [Plugin Repositories](plugin-repositories.md) — Creating and hosting custom plugin repositories
- [Development Server](dev-server.md) — Local development server for push-to-device testing
- [PIP & Dependencies](pip.md) — Using third-party Python packages inside plugins
- [Available Libraries](available-libraries.md) — Pre-bundled Python libraries in exteraGram runtime
- [Common Telegram Classes](common-source-classes.md) — Common Telegram / TL classes and references

## Документы для нейросетей (LLM) и ассистента
- [SKILL.md](SKILL.md) — готовый агентный навык (Skill) для Antigravity / Claude / Cursor / Windsurf с правилами, шаблонами и триггерами.
- [AI_CHEATSHEET.md](AI_CHEATSHEET.md) — компактная шпаргалка и системный промпт для нейросетей (шаблоны, правила, архитектура, частые ошибки).
- [FULL_AI_GUIDE.md](FULL_AI_GUIDE.md) — единая полная база знаний со всеми 35 статьями документации в одном файле для загрузки в контекст LLM.
