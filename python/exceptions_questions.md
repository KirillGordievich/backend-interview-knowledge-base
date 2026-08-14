# Python — Исключения: Вопросы

> Теория: [exceptions.md](exceptions.md)

---

**Q: Какова иерархия исключений в Python?**

`BaseException` — корень. От него наследуются: `SystemExit`, `KeyboardInterrupt`, `GeneratorExit`, `Exception`. Все "обычные" исключения наследуются от `Exception`. Поэтому `except Exception` ловит всё кроме системных (`SystemExit`, `KeyboardInterrupt`).

---

**Q: Почему нельзя писать `except:` или `except BaseException`?**

Они перехватывают `SystemExit` и `KeyboardInterrupt`, что мешает завершению программы через Ctrl+C или `sys.exit()`. Всегда используй `except Exception` для отлова "обычных" ошибок.

---

**Q: Что делает блок `else` в `try/except/else/finally`?**

`else` выполняется только если в `try` **не было** исключения. Позволяет отделить "нормальный" код от обработки ошибок:

```python
try:
    result = do_something()
except ValueError:
    handle_error()
else:
    process(result)   # только если не было ошибки
finally:
    cleanup()         # всегда
```

---

**Q: Когда выполняется `finally`?**

Всегда — даже если было исключение, даже если в `try` или `except` есть `return`. Единственное исключение: `os._exit()` или аварийное завершение процесса.

---

**Q: Что такое exception chaining (`from`)?**

Связывание исключений — оригинальная ошибка сохраняется как `__cause__`:

```python
try:
    int('abc')
except ValueError as e:
    raise RuntimeError('Parsing failed') from e
# RuntimeError: Parsing failed
# ... caused by ValueError: invalid literal for int()
```

`raise ... from None` — подавляет цепочку (скрывает оригинальное исключение).

---

**Q: Как создать своё исключение?**

Наследоваться от `Exception` (или его подкласса):

```python
class ValidationError(Exception):
    def __init__(self, field: str, message: str):
        self.field = field
        self.message = message
        super().__init__(f'{field}: {message}')
```

---

**Q: `raise` vs `raise from` — в чём разница?**

- `raise NewError()` — implicit chaining: оригинальное исключение сохраняется в `__context__`
- `raise NewError() from original` — explicit chaining: оригинальное в `__cause__`, traceback показывает "caused by"
- `raise NewError() from None` — подавляет цепочку

---

**Q: Что такое `ExceptionGroup` (Python 3.11+)?**

Группа исключений — позволяет обрабатывать несколько одновременных ошибок (например, из `TaskGroup`):

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fail1())
        tg.create_task(fail2())
except* ValueError as eg:    # except* ловит подгруппу по типу
    for e in eg.exceptions:
        print(e)
except* TypeError as eg:
    ...
```

---

**Q: Можно ли перехватить исключение и потом его перевыбросить?**

Да, `raise` без аргументов перевыбрасывает текущее исключение с оригинальным traceback:

```python
try:
    risky()
except Exception:
    log_error()
    raise  # перевыбрасывает, traceback не теряется
```

---

**Q: Что такое `warnings` и чем отличается от исключений?**

Warnings — предупреждения, которые по умолчанию не прерывают выполнение. Используются для deprecation, потенциальных проблем. Можно настроить поведение: игнорировать, превращать в ошибку, логировать.

```python
import warnings
warnings.warn('This is deprecated', DeprecationWarning)
```

---

**Q: Как ловить несколько типов исключений в одном `except`?**

Через tuple:

```python
except (ValueError, TypeError, KeyError) as e:
    handle(e)
```
