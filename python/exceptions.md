# Python — Исключения

## Основы

Исключение — объект, сигнализирующий об ошибке. Прерывает нормальный ход выполнения и поднимается по стеку вызовов до ближайшего обработчика.

```python
raise ValueError('неверное значение')        # экземпляр
raise ValueError                             # класс → вызовет конструктор без аргументов
```

### Иерархия исключений

```
BaseException
├── SystemExit           # sys.exit()
├── KeyboardInterrupt    # Ctrl+C
├── GeneratorExit        # закрытие генератора
└── Exception            # все "обычные" исключения
    ├── ValueError
    ├── TypeError
    ├── RuntimeError
    ├── OSError
    ├── ImportError
    ├── AttributeError
    ├── NameError
    ├── KeyError / IndexError  (LookupError)
    ├── StopIteration
    ├── ZeroDivisionError  (ArithmeticError)
    └── ...
```

### except: vs except Exception:

```python
# except: — ловит ВСЁ, включая системные сигналы
try:
    ...
except:          # поймает KeyboardInterrupt, SystemExit, MemoryError, ...
    pass         # пользователь не сможет выйти через Ctrl+C!

# except Exception: — ловит только "обычные" исключения
try:
    ...
except Exception:  # НЕ ловит KeyboardInterrupt, SystemExit, GeneratorExit
    pass           # системные сигналы работают нормально
```

**Рекомендация:** всегда использовать `except Exception` или конкретный тип. `except:` — только если нужно поймать буквально всё (финальный обработчик в сервере).

---

## Паттерны обработки

### try / except / else / finally

```python
try:
    result = risky_operation()
except ValueError as e:           # поймать конкретное исключение
    handle_value_error(e)
except (TypeError, KeyError) as e: # поймать несколько типов
    handle_other(e)
except Exception as e:             # поймать всё "обычное"
    log_unexpected(e)
    raise                          # перебросить
else:
    # выполняется ТОЛЬКО если исключений не было
    process(result)
finally:
    # выполняется ВСЕГДА — и при ошибке, и без
    cleanup()
```

**Правило:** обработчики расставлять от частного к общему — более специфичные `except` сверху, иначе они никогда не сработают.

### finally: когда выполняется

`finally` выполняется всегда — даже если в `try` или `except` есть `return`. Единственное исключение: `os._exit()` или аварийное завершение процесса.

### try / finally без except

Используется для гарантированного освобождения ресурсов без перехвата ошибок:

```python
lock.acquire()
try:
    do_work()
finally:
    lock.release()  # выполнится даже при ошибке

# Предпочтительнее через контекстный менеджер:
with lock:
    do_work()
```

### Блок else

Отделяет код, который может вызвать обрабатываемое исключение, от кода который должен выполниться только при успехе:

```python
try:
    f = open('file.txt')
except FileNotFoundError:
    print('файл не найден')
else:
    # здесь открытие прошло успешно
    # исключения здесь не перехватываются этим try
    data = f.read()
    f.close()
```

---

## Перебрасывание исключений

```python
try:
    1 / 0
except ZeroDivisionError:
    log_error()
    raise           # перебросить то же исключение без изменений

# Перебросить другое
try:
    connect()
except ConnectionError as e:
    raise RuntimeError('не удалось подключиться') from e
```

---

## Перехват нескольких типов исключений

```python
except (ValueError, TypeError, KeyError) as e:
    handle(e)
```

Передаётся кортеж типов — Python проверяет каждый по очереди. Предпочтительнее отдельных `except`-блоков, если логика обработки одинакова.

---

## Цепочки исключений

В Python 3 при исключении внутри `except` исходное сохраняется в `__context__` (неявная цепочка). Явное связывание через `from` сохраняет в `__cause__` и показывает "caused by" в traceback.

Разница между `raise` и `raise from`:
- `raise NewError()` — implicit chaining: оригинальное исключение в `__context__`
- `raise NewError() from original` — explicit chaining: оригинальное в `__cause__`, traceback показывает "caused by"
- `raise NewError() from None` — подавляет цепочку

Примеры:

```python
# Неявная цепочка (автоматически)
try:
    int('abc')
except ValueError:
    raise TypeError('ожидалось число')
# Output: During handling of the above exception, another exception occurred

# Явная цепочка (причина)
try:
    connect_db()
except OSError as e:
    raise DatabaseError('DB недоступна') from e
# Output: The above exception was the direct cause of...

# Подавить исходное исключение
try:
    risky()
except Exception:
    raise NewError('обёрнутая ошибка') from None
# Только NewError, исходное скрыто
```

---

## ExceptionGroup (Python 3.11+)

Группа исключений — позволяет обрабатывать несколько одновременных ошибок (например, из `asyncio.TaskGroup`). Синтаксис `except*` ловит подгруппу по типу:

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fail1())
        tg.create_task(fail2())
except* ValueError as eg:    # eg.exceptions — список ValueError
    for e in eg.exceptions:
        print(e)
except* TypeError as eg:
    ...
```

`ExceptionGroup` можно создавать вручную: `ExceptionGroup('msg', [err1, err2])`.

---

## Пользовательские исключения

```python
# Простое
class AppError(Exception):
    pass

# С дополнительными данными
class ValidationError(ValueError):
    def __init__(self, field, message):
        self.field = field
        super().__init__(f'{field}: {message}')

try:
    raise ValidationError('email', 'неверный формат')
except ValidationError as e:
    print(e.field)    # 'email'
    print(str(e))     # 'email: неверный формат'
```

**Конвенции:**
- Наследовать от `Exception` (не `BaseException`)
- Имя заканчивается на `Error`
- Создать базовый класс для ошибок своей библиотеки: `class MyLibError(Exception): pass`

---

## Предупреждения (warnings)

Для ситуаций, когда ошибки нет, но пользователя нужно уведомить (deprecated API, потенциальная проблема):

```python
import warnings

warnings.warn('Этот метод устарел, используйте new_method()', DeprecationWarning, stacklevel=2)
```

`stacklevel=2` — чтобы в выводе указывалось место вызова функции, а не сама функция с предупреждением.

Иерархия: `Warning → UserWarning, DeprecationWarning, RuntimeWarning, ...`

---

## Часто задаваемые вопросы

### Что произойдёт, если исключение не поймать

Поднимается по стеку вызовов до ближайшего обработчика. Если не найден — интерпретатор завершает программу и выводит traceback в `sys.stderr`.

Исключение: в деструкторе (`__del__`) — программа не завершается, выводится предупреждение.

### Можно ли поймать SyntaxError

Да — если ошибка возникает во время выполнения:
- в импортируемом модуле
- в строке, передаваемой в `eval()` или `exec()`

При ошибке в главном файле — нет, она возникает до запуска.
