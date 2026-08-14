# Python — ООП: Вопросы

> Теория: [oop.md](oop.md)

---

**Q: Чем отличается `__new__` от `__init__`?**

- `__new__` — создаёт объект (аллоцирует память), возвращает экземпляр. Вызывается до `__init__`.
- `__init__` — инициализирует уже созданный объект (заполняет атрибуты), ничего не возвращает.

`__new__` нужен когда: наследуешь immutable тип (`str`, `int`, `tuple`), реализуешь singleton, контролируешь создание экземпляра.

---

**Q: Что такое MRO и как Python определяет порядок вызова методов?**

MRO (Method Resolution Order) — порядок, в котором Python ищет методы при множественном наследовании. Используется алгоритм C3-линеаризация. Посмотреть: `MyClass.__mro__` или `MyClass.mro()`.

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass
# D.__mro__ → (D, B, C, A, object)
```

---

**Q: Что делает `super()` и как работает с множественным наследованием?**

`super()` вызывает метод следующего класса по MRO (не обязательно прямого родителя). При множественном наследовании — обходит всех по цепочке MRO, если каждый класс вызывает `super()`.

---

**Q: Что такое ABC (Abstract Base Class)?**

Абстрактный класс — нельзя создать экземпляр, пока все абстрактные методы не реализованы:

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

Shape()  # TypeError — нельзя инстанцировать
```

---

**Q: Что такое метакласс?**

Класс, который создаёт классы. По умолчанию — `type`. Метакласс контролирует создание класса через `__new__` и `__init__`. Используется для: валидации атрибутов класса, автоматической регистрации, изменения поведения класса.

```python
class Meta(type):
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        # модификация класса
        return cls

class MyClass(metaclass=Meta):
    pass
```

---

**Q: Как реализовать Singleton в Python?**

Несколько способов:
1. Через `__new__`
2. Через метакласс
3. Через декоратор
4. Через модуль (модуль — это естественный singleton)

```python
class Singleton:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

---

**Q: Что такое дескриптор?**

Объект с методами `__get__`, `__set__` и/или `__delete__`. Контролирует доступ к атрибуту класса. `property`, `classmethod`, `staticmethod` — встроенные дескрипторы.

- **Data descriptor** — есть `__set__` или `__delete__` (приоритет над `__dict__`)
- **Non-data descriptor** — только `__get__` (уступает `__dict__`)

---

**Q: Как работает `property`?**

`property` — дескриптор для управления доступом к атрибуту:

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError
        self._radius = value
```

---

**Q: `classmethod` vs `staticmethod` — в чём разница?**

- `@classmethod` — получает класс (`cls`) первым аргументом, может создавать экземпляры и обращаться к атрибутам класса
- `@staticmethod` — не получает ни `self`, ни `cls`, обычная функция в namespace класса

---

**Q: Что такое контекстный менеджер и как его создать?**

Объект с `__enter__` и `__exit__`. Используется с `with` для управления ресурсами. Два способа:

```python
# Через класс
class ManagedFile:
    def __enter__(self): ...
    def __exit__(self, exc_type, exc_val, exc_tb): ...

# Через декоратор
from contextlib import contextmanager
@contextmanager
def managed_file(path):
    f = open(path)
    try:
        yield f
    finally:
        f.close()
```

---

**Q: Что такое `__slots__` и зачем нужен?**

Заменяет `__dict__` фиксированным набором атрибутов. Экономит ~40-50% памяти, ускоряет доступ. Нельзя добавить произвольные атрибуты. Используй при миллионах экземпляров.

```python
class Point:
    __slots__ = ('x', 'y')
```

---

**Q: Чем отличается `_name` от `__name`?**

- `_name` — конвенция "приватное", технически доступно снаружи
- `__name` — name mangling: переименовывается в `_ClassName__name`. Защищает от случайного перекрытия в подклассах (не для "настоящей" приватности)

---

**Q: Что такое утиная типизация (duck typing)?**

"Если крякает как утка — значит утка." Объект подходит по типу если реализует нужный интерфейс (методы), независимо от иерархии наследования. Python предпочитает duck typing проверкам `isinstance`.

---

**Q: `__eq__` и `__hash__` — какая связь?**

Если переопределяешь `__eq__`, Python автоматически ставит `__hash__ = None` (объект становится unhashable). Если нужен hashable объект — переопредели оба. Правило: `a == b` → `hash(a) == hash(b)`.

---

**Q: Что такое dataclass?**

Декоратор `@dataclass` автоматически генерирует `__init__`, `__repr__`, `__eq__` и другие методы. Убирает boilerplate — не нужно вручную писать конструктор и сравнение для классов-контейнеров данных (DTO, конфиги, модели):

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)  # автоматический __init__
```

Параметры: `frozen=True` (immutable), `order=True` (сравнение), `slots=True` (Python 3.10+).

---

**Q: Чем `dataclass` отличается от `NamedTuple`?**

- `dataclass` — mutable (по умолчанию), класс, гибкий (default_factory, post_init)
- `NamedTuple` — immutable, tuple, hashable, можно распаковывать, легковесный
