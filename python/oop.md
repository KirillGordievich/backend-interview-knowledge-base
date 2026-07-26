# Python — ООП

## Абстрактные классы

### Что такое абстрактный класс

В Python нет интерфейсов как отдельной концепции, но есть **абстрактные классы** из модуля `abc`. Они позволяют объявить обязательный интерфейс для наследников — попытка создать экземпляр класса, не реализовавшего все абстрактные методы, вызовет `TypeError`.

```python
from abc import ABC, abstractmethod

class Shape(ABC):

    @abstractmethod
    def area(self) -> float:
        ...

    @abstractmethod
    def perimeter(self) -> float:
        ...

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius

    def area(self) -> float:
        return 3.14159 * self.radius ** 2

    def perimeter(self) -> float:
        return 2 * 3.14159 * self.radius

# Shape()   # TypeError: Can't instantiate abstract class
Circle(5)   # OK — все методы реализованы
```

---

### Декораторы abc

| Декоратор | Описание |
|---|---|
| `@abstractmethod` | Абстрактный метод |
| `@property` + `@abstractmethod` | Абстрактное свойство |
| `@staticmethod` + `@abstractmethod` | Абстрактный статический метод |
| `@classmethod` + `@abstractmethod` | Абстрактный метод класса |

```python
from abc import ABC, abstractmethod

class Base(ABC):

    @abstractmethod
    def regular(self):
        ...

    @property
    @abstractmethod
    def value(self):
        ...

    @staticmethod
    @abstractmethod
    def static_method():
        ...

    @classmethod
    @abstractmethod
    def class_method(cls):
        ...
```

---

### Абстрактный класс vs ABC

Два равнозначных способа объявить абстрактный класс:

```python
from abc import ABC, abstractmethod, ABCMeta

# Способ 1: наследование от ABC
class MyABC(ABC):
    @abstractmethod
    def method(self): ...

# Способ 2: явный metaclass (нужен при множественном наследовании с другим metaclass)
class MyABC2(metaclass=ABCMeta):
    @abstractmethod
    def method(self): ...
```

---

### register() — виртуальные подклассы

`ABC.register(cls)` позволяет объявить класс подклассом абстрактного, не наследуясь от него. `isinstance` и `issubclass` будут возвращать `True`, но реализация методов не проверяется.

```python
class MySeq(ABC):
    @abstractmethod
    def __len__(self): ...

MySeq.register(list)
isinstance([], MySeq)  # True
```

---

## Метаклассы

### type — метакласс по умолчанию

`type` — это метакласс, который Python использует для создания всех классов. Класс — это объект типа `type`.

```python
# Эти записи эквивалентны:
class MyClass:
    x = 42

MyClass = type('MyClass', (object,), {'x': 42})

type(int)      # <class 'type'>
type(MyClass)  # <class 'type'>
```

`type(name, bases, namespace)`:
- `name` — имя класса
- `bases` — кортеж базовых классов
- `namespace` — словарь атрибутов

### Пользовательский метакласс

Метакласс позволяет перехватить и изменить процесс создания класса:

```python
class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    pass

db1 = Database()
db2 = Database()
db1 is db2  # True
```

---

## MRO (Method Resolution Order)

### Что такое MRO

Порядок, в котором Python ищет методы при множественном наследовании. Определяется алгоритмом **C3-линеаризации**.

```python
class A:
    def method(self): print('A')

class B(A):
    def method(self): print('B')

class C(A):
    def method(self): print('C')

class D(B, C):
    pass

D().method()    # 'B' — ищем по MRO
D.__mro__       # (D, B, C, A, object)
D.mro()         # то же в виде списка
```

**Правила C3:**
1. Класс идёт перед своими родителями
2. Порядок родителей сохраняется
3. Если класс встречается в нескольких местах — берётся последнее

### super() и MRO

`super()` вызывает следующий класс в MRO, не обязательно прямого родителя:

```python
class B(A):
    def method(self):
        super().method()  # вызовет следующий по MRO от B
        print('B')

class D(B, C):
    def method(self):
        super().method()  # вызовет B.method (следующий в MRO)
        print('D')
```

---

## __new__ vs __init__

| | `__new__` | `__init__` |
|---|---|---|
| Когда вызывается | До создания объекта | После создания |
| Что делает | Выделяет память, возвращает новый объект | Инициализирует атрибуты |
| Тип | Статический метод | Обычный метод |
| Возвращает | Экземпляр класса | None |

```python
class MyClass:
    def __new__(cls, *args, **kwargs):
        print('__new__ вызван')
        instance = super().__new__(cls)  # создаём объект
        return instance

    def __init__(self, value):
        print('__init__ вызван')
        self.value = value

obj = MyClass(42)
# __new__ вызван
# __init__ вызван
```

**Практический пример — Singleton через `__new__`:**

```python
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

**Когда нужен `__new__`:**
- Создание Singleton
- Наследование от неизменяемых типов (`int`, `str`, `tuple`)
- Управление пулом объектов

---

## Дескрипторы

Дескриптор — объект, который управляет доступом к атрибуту другого класса через методы `__get__`, `__set__`, `__delete__`.

```python
class PositiveNumber:
    def __set_name__(self, owner, name):
        self.name = name              # имя атрибута в классе-хозяине

    def __get__(self, instance, owner):
        if instance is None:
            return self               # обращение через класс
        return instance.__dict__.get(self.name)

    def __set__(self, instance, value):
        if value < 0:
            raise ValueError(f'{self.name} должен быть положительным')
        instance.__dict__[self.name] = value

class Product:
    price = PositiveNumber()
    quantity = PositiveNumber()

p = Product()
p.price = 100    # OK
p.price = -5     # ValueError: price должен быть положительным
```

**Типы дескрипторов:**
- **Data descriptor** — реализует `__set__` или `__delete__` → приоритет выше `__dict__` экземпляра
- **Non-data descriptor** — только `__get__` → приоритет ниже `__dict__` экземпляра

**Встроенные дескрипторы:** `property`, `staticmethod`, `classmethod`.

**Когда использовать:**
- Валидация значений атрибутов
- Ленивое вычисление (кэширование через `cached_property`)
- Логирование доступа к атрибутам

---

## Циклические зависимости

### Проблема

Если `A` зависит от `B`, `B` от `C`, а `C` от `A` — импорты падают с `ImportError` или AttributeError.

### Решения

**1. Перенести общую логику в третий модуль:**
```
A → D ← B ← C
         ↑
         A
```

**2. Отложенный импорт (внутри функции/метода):**
```python
# a.py
def get_b():
    from b import B  # импортируется только при вызове функции
    return B()
```

**3. Внедрение зависимостей (Dependency Injection):**
```python
class Service:
    def __init__(self, repository):  # получает зависимость снаружи
        self.repo = repository       # не импортирует напрямую
```

### Как предотвратить

- **SRP (Single Responsibility Principle):** один класс — одна ответственность
- **DIP (Dependency Inversion Principle):** зависеть от абстракций, не от конкретных классов
- **Dependency Injection:** зависимости передаются через конструктор

---

## Атрибуты объектов

### Как получить список атрибутов объекта

`dir(obj)` — список всех атрибутов и методов (включая унаследованные).
`obj.__dict__` — словарь `{имя → значение}` только для атрибутов **экземпляра** (не методов класса).

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
p.__dict__   # {'x': 1, 'y': 2}
dir(p)       # ['__class__', ..., 'x', 'y']
```

---

## Магические методы (dunder methods)

Методы с двойным подчёркиванием в начале и конце (`__name__`). Вызываются неявно — встроенными функциями или синтаксическими конструкциями. Никогда не вызывайте их напрямую (за исключением `super().__init__()`).

### Основные группы

| Группа | Методы |
|---|---|
| Жизненный цикл | `__new__`, `__init__`, `__del__` |
| Строковое представление | `__str__`, `__repr__`, `__format__` |
| Сравнение | `__eq__`, `__ne__`, `__lt__`, `__le__`, `__gt__`, `__ge__` |
| Арифметика | `__add__`, `__sub__`, `__mul__`, `__truediv__`, `__floordiv__`, `__mod__`, `__pow__` |
| Правосторонняя арифметика | `__radd__`, `__rsub__`, ... |
| Составное присваивание | `__iadd__`, `__isub__`, ... |
| Контейнер | `__len__`, `__getitem__`, `__setitem__`, `__delitem__`, `__contains__` |
| Итерация | `__iter__`, `__next__` |
| Вызываемый объект | `__call__` |
| Контекстный менеджер | `__enter__`, `__exit__` |
| Хеширование | `__hash__` |
| Атрибуты | `__getattr__`, `__setattr__`, `__delattr__`, `__getattribute__` |
| Дескриптор | `__get__`, `__set__`, `__delete__` |

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __repr__(self):               # repr(v) → используется в отладке
        return f'Vector({self.x}, {self.y})'

    def __str__(self):                # str(v) → используется в print()
        return f'({self.x}, {self.y})'

    def __add__(self, other):         # v1 + v2
        return Vector(self.x + other.x, self.y + other.y)

    def __len__(self):                # len(v)
        return 2

    def __eq__(self, other):          # v1 == v2
        return self.x == other.x and self.y == other.y
```

### __str__ vs __repr__

- `__repr__` — точное представление для разработчика. Должно позволять воссоздать объект: `eval(repr(obj)) == obj`
- `__str__` — читаемое для пользователя. Если не определён — используется `__repr__`
- `print(obj)` вызывает `__str__`, `repr(obj)` — `__repr__`

---

## Diamond problem

При ромбовидном наследовании MRO определяет порядок вызовов. Алгоритм C3 гарантирует, что каждый класс встречается в MRO только один раз (берётся последнее вхождение при DFLR-обходе).

```python
class D:
    attr = 3

class B(D):
    pass

class C(D):
    attr = 1

class A(B, C):
    pass

# DFLR = [A, B, D, C, D]  → убираем дубликат D кроме последнего
# MRO  = [A, B, C, D, object]
A().attr     # 1 (C.attr) — не 3 (D.attr)!
A.__mro__    # (A, B, C, D, object)
```

---

## Миксины (Mixins)

Паттерн: небольшой класс-помощник, добавляемый в цепочку множественного наследования. Реализует одну конкретную функциональность, не предназначен для самостоятельного использования.

```python
class TimestampMixin:
    """Добавляет метод now() любому классу."""
    def now(self):
        import datetime
        return datetime.datetime.utcnow()

class JsonMixin:
    """Добавляет метод to_json()."""
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class User(TimestampMixin, JsonMixin):
    def __init__(self, name):
        self.name = name

u = User('Alice')
u.now()      # datetime(...)
u.to_json()  # '{"name": "Alice"}'
```

По соглашению в имени миксина всегда есть слово `Mixin`. Технически это обычный класс — Python не имеет специального синтаксиса для миксинов.

---

## Контекстный менеджер

Объект, управляющий ресурсами через оператор `with`. Гарантирует выполнение завершающего кода даже при исключении.

Должен реализовывать:
- `__enter__(self)` — вход в блок `with`, возвращаемое значение доступно через `as`
- `__exit__(self, exc_type, exc_val, exc_tb)` — выход из блока. Если вернуть `True` — исключение подавляется

```python
class ManagedFile:
    def __init__(self, path):
        self.path = path

    def __enter__(self):
        self.file = open(self.path)
        return self.file

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.file.close()
        return False  # не подавлять исключения

with ManagedFile('data.txt') as f:
    data = f.read()
```

Через `contextlib.contextmanager` — декораторный способ:

```python
from contextlib import contextmanager

@contextmanager
def managed_file(path):
    f = open(path)
    try:
        yield f       # значение после yield → переменная as
    finally:
        f.close()     # выполнится всегда

with managed_file('data.txt') as f:
    data = f.read()
```

**Выходить из контекстного менеджера нужно как можно быстрее** — пока идёт блок `with`, ресурс заблокирован.

---

## Идентичность vs равенство

```python
object() == object()   # False
object() is object()   # False
```

По умолчанию `==` сравнивает по `id` (адрес в памяти), если не переопределён `__eq__`. Это значит, что два разных объекта всегда неравны даже при одинаковом содержимом.

```python
a = [1, 2, 3]
b = [1, 2, 3]
a == b   # True  — сравнение по значению (list переопределяет __eq__)
a is b   # False — разные объекты в памяти

c = a
a is c   # True  — одна и та же ссылка
```

---

## __slots__

По умолчанию атрибуты экземпляра хранятся в `__dict__` (словарь). `__slots__` заменяет словарь фиксированным набором слотов.

```python
class Point:
    __slots__ = ('x', 'y')  # только x и y, __dict__ не создаётся

    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
p.z = 3       # AttributeError — нет такого слота
p.__dict__    # AttributeError — нет __dict__
```

**Когда использовать:**
- Миллионы экземпляров — экономия памяти (~40-50% меньше чем с __dict__)
- Критична производительность — доступ к слоту быстрее чем к __dict__

**Ограничения:**
- Нельзя добавить произвольный атрибут
- Не работает `__getattr__` / `__setattr__` по умолчанию
- При наследовании: родитель без `__slots__` — дочерний класс всё равно получит `__dict__`

---

## Name mangling (_x и __x)

### Один подчёркивание (`_name`)

Конвенция: атрибут/метод для внутреннего использования. Технически доступен снаружи, но IDE предупредит.

```python
class Foo:
    def __init__(self):
        self._internal = 42   # "не трогай снаружи"

Foo()._internal  # 42 — доступен, но по конвенции не нужно
```

### Два подчёркивания (`__name`)

**Name mangling** — интерпретатор переименовывает атрибут в `_ClassName__name`. Защищает от случайного перекрытия в подклассах.

```python
class Foo:
    def __init__(self):
        self.__private = 42

f = Foo()
f.__private         # AttributeError
f._Foo__private     # 42 — всё ещё доступен, но имя изменено
```

Используется не для "настоящей" приватности, а чтобы избежать конфликта имён при наследовании.

---

## Утиная типизация (Duck Typing)

> "Если это выглядит как утка, плавает как утка и крякает как утка — это, вероятно, и есть утка."

Объект считается подходящим типом, если он реализует нужный интерфейс (методы/атрибуты), независимо от его реального типа или иерархии наследования.

```python
class Duck:
    def quack(self): return 'Quack!'
    def walk(self):  return 'Waddle'

class Person:
    def quack(self): return 'I am quacking like a duck!'
    def walk(self):  return 'Walking like a duck'

def make_it_quack(duck):  # не проверяем isinstance!
    return duck.quack()

make_it_quack(Duck())    # 'Quack!'
make_it_quack(Person())  # 'I am quacking like a duck!'
```

Python предпочитает утиную типизацию явным проверкам `isinstance`. Формализация — через `Protocol` из `typing` (см. [typing.md](typing.md)).
