# Python — Модули и пакеты

## Модуль

Файл с исходным кодом Python (`.py`). Позволяет разбивать программу на независимые, переиспользуемые части.

### __name__ и __main__

Каждый модуль имеет атрибут `__name__`. При импорте он равен имени файла без расширения. При прямом запуске — `"__main__"`.

```python
# mymodule.py
print(__name__)

# python mymodule.py  → '__main__'
# import mymodule     → 'mymodule'
```

Паттерн для кода, который должен выполняться только при прямом запуске:

```python
def main():
    print('Запуск основной логики')

if __name__ == '__main__':
    main()
```

---

## Как Python ищет модули при импорте

При `import foo` интерпретатор ищет модуль в следующем порядке:

1. **sys.modules** — кэш уже загруженных модулей (быстрая проверка)
2. **Встроенные модули** (built-in): `sys`, `os`, `builtins` и т.д.
3. **sys.path** — список директорий и архивов:
   - директория со скриптом (или текущая директория)
   - `PYTHONPATH` (переменная окружения)
   - стандартная библиотека
   - `site-packages` (установленные пакеты)

```python
import sys
print(sys.path)  # список путей поиска

# Добавить путь динамически (не рекомендуется в production)
sys.path.insert(0, '/my/custom/path')
```

Повторный импорт одного и того же модуля не перечитывает файл — возвращается объект из `sys.modules`.

---

## Пакет

Директория с файлом `__init__.py`. Позволяет организовывать модули иерархически.

```
mypackage/
    __init__.py       # инициализация пакета, может быть пустым
    module_a.py
    module_b.py
    subpackage/
        __init__.py
        module_c.py
```

```python
import mypackage.module_a            # импорт модуля из пакета
from mypackage import module_a       # то же, короче
from mypackage.module_a import func  # импорт конкретного имени
```

`__init__.py` выполняется при первом импорте пакета. Можно использовать для:
- переэкспорта имён (`from .module_a import PublicClass`)
- ленивой загрузки подмодулей
- инициализации ресурсов пакета

```python
# mypackage/__init__.py
from .module_a import PublicClass  # теперь mypackage.PublicClass работает
```

**Namespace packages (Python 3.3+):** пакеты без `__init__.py`. Позволяют разбить один пакет по нескольким директориям.

---

## from package import item

```python
import package.module     # item должен быть модулем или пакетом
from package import item  # item может быть модулем, классом, функцией, переменной
```

```python
# Абсолютный импорт (рекомендуется)
from mypackage.module_a import MyClass

# Относительный импорт (только внутри пакета)
from . import module_b        # из текущего пакета
from .. import module_c       # из родительского пакета
from .module_b import helper  # из соседнего модуля
```

---

## __all__

Управляет тем, что экспортируется при `from module import *`:

```python
# mymodule.py
__all__ = ['PublicClass', 'public_func']  # только эти имена

class PublicClass: ...
def public_func(): ...
def _private_func(): ...  # не попадёт в import *
```

---

## Перезагрузка модуля

Модуль загружается один раз и кэшируется. Для принудительной перезагрузки:

```python
import importlib
import mymodule

importlib.reload(mymodule)  # перечитает файл и выполнит заново
```

Используется редко — в основном в REPL при разработке.
