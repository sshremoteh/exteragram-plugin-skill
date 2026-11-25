# Шпаргалка и системный промпт для нейросетей (exteraGram Plugins)

Этот документ предназначен для передачи языковым моделям (LLM) в качестве контекста или системного промпта при разработке плагинов для клиента **exteraGram** (Android).

---

## 1. Базовые правила и ограничения

1. **Среда выполнения**:
   - Python 3.11 через **Chaquopy v16** внутри Android-приложения.
   - Минимальная версия приложения: **exteraGram 12.5.1+**.
   - Версия SDK: **1.4.4.3+**.
2. **Парсинг метаданных**:
   - Метаданные парсятся **статически через AST**. Нельзя использовать вычисляемые выражения, функции или импорты для заполнения метаданных. Только строковые литералы на верхнем уровне файла.
3. **Регистрация хуков**:
   - Объявление метода хука (например, `on_send_message_hook`) само по себе **НЕ регистрирует его**.
   - Его обязательно нужно явно зарегистрировать в `on_plugin_load(self)` вызовом соответствующего метода (например, `self.add_on_send_message_hook()`).
4. **Потоки (Threading)**:
   - Хуки и UI вызываются в главном UI-потоке Android. **Запрещено** делать долгие операции (HTTP-запросы, чтение больших файлов) напрямую в хуках без `run_on_queue(...)` из `client_utils` или потоков, иначе приложение зависнет (ANR).
   - Любое обращение к UI (диалоги, тосты, бюллетени) из фонового потока должно оборачиваться в `run_on_ui_thread(...)`.

---

## 2. Шаблон одиночного плагина (Single-file Plugin)

```python
from typing import Any, List
from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Header, Input, Switch, Text
from client_utils import run_on_queue
from android_utils import run_on_ui_thread

# Метаданные обязательно константами в начале файла:
__id__ = "my_custom_plugin"
__name__ = "My Custom Plugin"
__description__ = "Описание того, что делает плагин"
__author__ = "Author Name"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"

DEFAULT_PREFIX = "."

class MyCustomPlugin(BasePlugin):
    def on_plugin_load(self):
        # Обязательная регистрация хуков
        self.add_on_send_message_hook()
        self.logger.info("Plugin loaded successfully")

    def on_plugin_unload(self):
        self.logger.info("Plugin unloaded")

    def create_settings(self) -> List[Any]:
        # Настройки в меню плагина
        return [
            Header(text="Основные настройки"),
            Input(
                key="cmd_prefix",
                text="Префикс команд",
                default=DEFAULT_PREFIX,
                subtext="Символ для вызова команд плагина",
                icon="msg_edit",
            ),
            Switch(
                key="enabled_feature",
                text="Включить функцию",
                default=True,
                icon="msg_cmd",
            ),
            Text(
                text="Информация",
                subtext="Версия: 1.0.0",
                icon="msg_info",
            ),
        ]

    def on_send_message_hook(self, account: int, params: Any) -> HookResult:
        # Проверяем наличие текстового сообщения
        if not hasattr(params, "message") or not isinstance(params.message, str):
            return HookResult()

        text = params.message.strip()
        prefix = self.get_setting("cmd_prefix", DEFAULT_PREFIX)

        if text.startswith(f"{prefix}ping"):
            params.message = "Pong!"
            return HookResult(strategy=HookStrategy.MODIFY, params=params)

        # Если ничего не меняем:
        return HookResult()
```

---

## 3. Стратегии `HookResult`

При перехвате событий возвращается объект `HookResult`:
- `HookResult()` — не вносить изменений, пропустить исходное действие как есть.
- `HookResult(strategy=HookStrategy.MODIFY, params=params)` — применить измененные параметры.
- `HookResult(strategy=HookStrategy.CANCEL)` — прервать/отменить действие (сообщение не отправится, запрос не уйдет).

---

## 4. Ключевые модули SDK

| Модуль | Назначение | Ключевые функции / классы |
| :--- | :--- | :--- |
| `base_plugin` | Базовый класс плагина, хуки, меню, Xposed | `BasePlugin`, `HookResult`, `HookStrategy`, `MethodHook`, `MethodReplacement` |
| `client_utils` | Работа с сообщениями и клиентом | `send_text`, `send_message`, `edit_message`, `run_on_queue`, `get_account_instance` |
| `android_utils` | Интеграция с Android | `run_on_ui_thread`, `show_toast`, `copy_to_clipboard`, `log` |
| `ui.settings` | Компоненты экрана настроек | `Header`, `Input`, `Switch`, `Selector`, `Text`, `Button`, `Check` |
| `ui.alert` | Всплывающие диалоги Telegram | `AlertDialogBuilder` |
| `ui.bulletin` | Всплывающие уведомления (Bulletins) | `BulletinHelper` (`show_info`, `show_success`, `show_error`) |
| `extera_utils.text_formatting` | Форматирование текста Telegram | `parse_text` |
| `extera_utils.classes` | Генерация Java-прокси / подклассов | `Base`, `java_subclass`, `joverride`, `jfield` |
| `hook_utils` | Рефлексия Java | `find_class`, `get_private_field`, `call_method` |
| `file_utils` | Файловая система плагина | `get_plugin_data_dir`, `read_file`, `write_file` |

---

## 5. Фреймворк Elyx (для модульных плагинов)

Если проект состоит из нескольких файлов или использует внешние зависимости:
- Структура проекта:
  - `elyx.toml` — метаданные и настройки сборки.
  - `src/main.py` — точка входа (`class Plugin(BasePlugin)`).
  - `src/modules/` — локальные подмодули.
  - `assets/` — иконки и статика.
  - `locales/` — переводы (`ru.json`, `en.json`).
- Сборка и управление: утилита `elyx` CLI (`elyx build`, `elyx run`, `elyx add`).

---

## 6. Частые ошибки, которых нейросеть должна избегать

1. **Не вычислять метаданные**: Никаких `__version__ = get_version()` или `__description__ = f"..."`!
2. **Не забывать `self.add_on_*_hook()`**: Регистрация в `on_plugin_load` строго обязательна.
3. **Не блокировать UI**: Сетевые запросы (`urllib`, `requests`) оборачивать в `run_on_queue(...)`.
4. **Проверять тип сообщений**: В `on_send_message_hook` параметр `params.message` может быть `None` (например, при отправке только фото или стикера). Всегда проверяйте `isinstance(getattr(params, "message", None), str)`.
5. **Безопасное получение настроек**: Всегда указывать дефолтное значение: `self.get_setting("key", default_val)`.
