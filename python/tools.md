# Python — Инструменты разработки

## Линтеры

Линтер — программа, которая статически анализирует код и сообщает о нарушениях стандартов, потенциальных ошибках и проблемах со стилем. Не запускает код.

**Зачем:**
- Единый стандарт кода в команде
- Нахождение ошибок до запуска
- Автоматизируется в CI/CD

### Популярные линтеры

| Инструмент | Что делает |
|---|---|
| **Ruff** | Линтер + форматтер, написан на Rust, в 100x быстрее flake8 |
| **flake8** | PEP8 + базовые ошибки, экосистема плагинов |
| **pylint** | Глубокий анализ, много правил, медленнее |
| **black** | Форматтер (не линтер), не настраивается — единый стиль |
| **isort** | Сортировка импортов |
| **mypy / Pyright** | Проверка типов |

### Конфигурация (ruff)

```toml
# pyproject.toml
[tool.ruff]
line-length = 88
select = ["E", "F", "I"]  # pycodestyle, pyflakes, isort

[tool.ruff.lint]
ignore = ["E501"]
```

```bash
ruff check .          # проверить
ruff check --fix .    # исправить автоматически
ruff format .         # форматировать
```

---

## Poetry

Менеджер зависимостей и пакетов. Управляет виртуальным окружением, зависимостями и публикацией пакетов.

```bash
poetry new my-project      # создать новый проект
poetry init                # добавить poetry в существующий проект
poetry add requests        # добавить зависимость
poetry add pytest --group dev  # зависимость для разработки
poetry install             # установить все зависимости из lock-файла
poetry run python app.py   # запустить в виртуальном окружении
poetry shell               # активировать окружение
poetry update              # обновить зависимости
```

**Файлы:**
- `pyproject.toml` — конфигурация проекта и зависимости
- `poetry.lock` — зафиксированные версии всех зависимостей (коммитить в репо)

---

## uv

Современная замена pip + venv + pip-tools. Написан на Rust — значительно быстрее pip.

```bash
uv init my-project        # создать проект
uv add requests           # добавить зависимость
uv add pytest --dev       # dev-зависимость
uv install                # установить зависимости
uv run python app.py      # запустить в виртуальном окружении
uv sync                   # синхронизировать окружение с lock-файлом
uv pip install requests   # использовать как замену pip
```

**Преимущества перед poetry:**
- Скорость (Rust + глобальный кэш пакетов)
- Совместимость с существующими `requirements.txt`
- Не требует отдельного Python для работы самого uv

---

## pre-commit

Хуки, которые запускаются автоматически перед каждым коммитом.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.9.0
    hooks:
      - id: mypy
```

```bash
pre-commit install        # установить хуки
pre-commit run --all-files  # запустить вручную
```
