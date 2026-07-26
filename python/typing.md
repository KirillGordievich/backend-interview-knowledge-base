# Python — Type Hints

## Что такое type hints

Type hints — способ явно указать ожидаемые типы. Python остаётся динамически типизированным — аннотации **не проверяются в рантайме** (если не использовать специальные инструменты или `typing.get_type_hints()`).

**Зачем использовать:**
- Статические анализаторы (mypy, Pyright) находят ошибки до запуска
- IDE точнее автодополняет и подсвечивает ошибки
- Документация прямо в коде

```python
def greet(name: str) -> str:
    return 'Hello, ' + name
```

---

## Базовые аннотации

```python
x: int = 5
s: str = 'hello'
b: bool = True
n: None = None

# Коллекции — Python 3.9+ поддерживает встроенные дженерики
lst: list[int]       = [1, 2, 3]
tpl: tuple[int, str] = (1, 'a')
dct: dict[str, int]  = {'a': 1}
st:  set[str]        = {'x', 'y'}

# Кортеж произвольной длины
tpl2: tuple[int, ...]  = (1, 2, 3, 4)

# До Python 3.9 нужен импорт из typing
from typing import List, Dict  # List[int], Dict[str, int]
```

**list[int] vs List[int]:** предпочтительнее `list[int]` (Python 3.9+) — не требует импорта, короче. `List[int]` — для совместимости со старым кодом.

---

## Any, object, type

```python
from typing import Any

def f1(x: Any) -> Any: ...   # совместим с ЛЮБЫМ типом, проверка выключена
def f2(x: object) -> None: ...  # базовый класс всего, только .method общие
def f3(x: type) -> None: ...    # принимает класс (тип), не экземпляр
```

| | `Any` | `object` | `type` |
|---|---|---|---|
| Что принимает | всё | экземпляры любого типа | только классы |
| Проверка | выключена | строгая (только методы object) | принимает классы |
| Пример | legacy код | базовые операции | фабрики, метаклассы |

```python
x: Any    = 42;      x.whatever()   # OK для mypy (проверка выключена)
x: object = 42;      x.whatever()   # Error — нет такого метода у object
x: type   = int;     x()            # OK — int это класс
x: type   = 42       # Error — 42 не класс

# Any vs object — популярный вопрос:
# Any совместим с ЛЮБЫМ типом в обе стороны
# object совместим только как базовый тип (нельзя присвоить object → int)
def accepts_int(x: int) -> None: ...
accepts_int(Any())    # OK
accepts_int(object()) # Error — object не является int
```

---

## Optional и Union

```python
from typing import Optional, Union

# Optional[X] == Union[X, None] == X | None
def find(id: int) -> Optional[str]: ...
def find(id: int) -> str | None: ...   # 3.10+, предпочтительнее

# Union — один из нескольких типов
def process(data: Union[str, bytes]) -> None: ...
def process(data: str | bytes) -> None: ...    # 3.10+
```

**Optional[int] vs int | None:** семантически одинаковы. `X | None` — современный синтаксис (3.10+), предпочтителен в новом коде.

---

## Literal

Ограничивает значения конкретным набором литералов:

```python
from typing import Literal

def request(method: Literal['GET', 'POST', 'PUT', 'DELETE']) -> None: ...

request('GET')    # OK
request('PATCH')  # Error — не входит в Literal

# Полезно для строковых флагов
Mode = Literal['r', 'w', 'rb', 'wb']
def open_file(path: str, mode: Mode) -> None: ...
```

---

## TypeVar и Generic

`TypeVar` — переменная типа, позволяет выражать полиморфизм:

```python
from typing import TypeVar, Generic

T = TypeVar('T')

# Функция сохраняет тип: если дали list[int] — вернёт int
def first(items: list[T]) -> T:
    return items[0]

first([1, 2, 3])    # mypy знает: возвращает int
first(['a', 'b'])   # mypy знает: возвращает str
```

### Generic класс

```python
class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

s: Stack[int] = Stack()
s.push(42)   # OK
s.push('x')  # Error
```

### bound — ограничение TypeVar

```python
from typing import TypeVar
from collections.abc import Sized

# T может быть только Sized или его подклассом
T = TypeVar('T', bound=Sized)

def length(x: T) -> int:
    return len(x)

length([1, 2])   # OK
length(42)       # Error — int не Sized
```

### Constrained TypeVar

Ограничивает перечислением конкретных типов:

```python
# T может быть только int или str — не их подклассами
AnyStr = TypeVar('AnyStr', int, str)

def double(x: AnyStr) -> AnyStr:
    return x * 2

# bound vs constrained:
# bound=X  → T может быть X или любым подклассом X
# T(int, str) → T может быть ТОЛЬКО int или str
```

---

## Коллекции из collections.abc

Для аннотаций предпочтительно использовать абстрактные типы из `collections.abc` — они принимают любую совместимую реализацию:

```python
from collections.abc import (
    Iterable,    # __iter__
    Iterator,    # __iter__ + __next__
    Generator,   # Iterator + send/throw/close
    Sequence,    # Iterable + __len__ + __getitem__
    Collection,  # Sized + Iterable + Container
    Container,   # __contains__
    Mapping,     # dict-подобный, read-only
    MutableMapping,  # dict-подобный, изменяемый
)
```

| Тип | Методы | Примеры |
|---|---|---|
| `Container` | `__contains__` | list, set, dict |
| `Iterable` | `__iter__` | list, str, генератор |
| `Iterator` | `__iter__`, `__next__` | объект-итератор |
| `Generator` | Iterator + `send`, `throw`, `close` | функция с yield |
| `Sequence` | `__len__`, `__getitem__` | list, tuple, str |
| `Collection` | Sized + Iterable + Container | большинство коллекций |
| `Mapping` | `__getitem__`, `__len__`, `__iter__` | dict (read-only) |
| `MutableMapping` | Mapping + `__setitem__`, `__delitem__` | dict (writable) |

```python
# Предпочитайте Iterable вместо list в параметрах — более гибко
def process(items: Iterable[int]) -> int:  # принимает list, tuple, генератор
    return sum(items)

# Mapping вместо dict — принимает dict и любой dict-like объект
def render(config: Mapping[str, str]) -> str: ...
```

### Разница Iterator и Iterable

```python
# Iterable — можно получить итератор (может итерироваться много раз)
# Iterator — уже является итератором (одноразовый, помнит позицию)

lst = [1, 2, 3]       # Iterable — не Iterator
it = iter(lst)        # Iterator

isinstance(lst, Iterable)  # True
isinstance(lst, Iterator)  # False
isinstance(it, Iterator)   # True
isinstance(it, Iterable)   # True (Iterator наследует Iterable)
```

### Типизация генератора

`Generator[YieldType, SendType, ReturnType]`:

```python
from collections.abc import Generator

def counter(start: int) -> Generator[int, None, str]:
    # Yields: int, Send: None (не ожидает send), Return: str
    i = start
    while i < 10:
        yield i
        i += 1
    return 'done'

# Если только yield и не нужен send/return — используйте Iterator[int]
from collections.abc import Iterator

def count_up(n: int) -> Iterator[int]:
    for i in range(n):
        yield i
```

---

## Callable

```python
from collections.abc import Callable

# Callable[[param_types], return_type]
f: Callable[[int, str], bool]

# Произвольные аргументы
f: Callable[..., int]  # любые аргументы, возвращает int

def apply(func: Callable[[int, int], int], a: int, b: int) -> int:
    return func(a, b)

# Callback с произвольной сигнатурой
Handler = Callable[..., None]
```

---

## Protocol vs ABC

```python
from typing import Protocol
from abc import ABC, abstractmethod

# ABC — номинальная типизация: нужно явно наследоваться
class Drawable(ABC):
    @abstractmethod
    def draw(self) -> None: ...

# Protocol — структурная типизация: достаточно иметь нужные методы
class Drawable(Protocol):
    def draw(self) -> None: ...

class Circle:             # не наследует Protocol
    def draw(self) -> None:
        print('circle')

def render(d: Drawable) -> None:
    d.draw()

render(Circle())  # OK с Protocol, Error с ABC
```

| | `Protocol` | `ABC` |
|---|---|---|
| Типизация | Структурная (duck typing) | Номинальная |
| Наследование | Не нужно | Обязательно |
| Проверка | Только статическая (mypy) | Runtime `isinstance` |
| Когда | Не контролируете классы | Контролируете иерархию |

---

## TypedDict

Строгая типизация словарей:

```python
from typing import TypedDict, Required, NotRequired

class Movie(TypedDict):
    title: str
    year: int
    rating: float

# NotRequired — необязательные поля (Python 3.11+, или через typing_extensions)
class Config(TypedDict):
    host: str
    port: int
    timeout: NotRequired[float]   # можно не передавать
    debug: NotRequired[bool]

# TypedDict vs dataclass:
# TypedDict — это словарь с проверкой типов ключей, нет методов, нет default_factory
# dataclass — настоящий класс с методами, __init__, __repr__ и т.д.
```

---

## NewType

Создаёт различимый тип — алиас, который mypy считает **отдельным типом**:

```python
from typing import NewType

UserId = NewType('UserId', int)
PostId = NewType('PostId', int)

def get_user(user_id: UserId) -> None: ...

uid = UserId(42)
pid = PostId(42)

get_user(uid)   # OK
get_user(pid)   # Error — PostId != UserId, хотя оба int!
get_user(42)    # Error — int != UserId

# Зачем: предотвращает случайную путаницу разных ID
# В runtime NewType — просто identity function, нет overhead
```

---

## TypeAlias

Явно объявляет псевдоним типа (иначе mypy может считать его переменной):

```python
from typing import TypeAlias   # Python 3.10+

Vector: TypeAlias = list[float]
Matrix: TypeAlias = list[list[float]]

# Python 3.12+ — новый синтаксис
type Vector = list[float]
type Matrix = list[list[float]]
```

---

## Self

Типизация методов, возвращающих `self` — работает корректно при наследовании:

```python
from typing import Self

class Builder:
    def set_name(self, name: str) -> Self:  # возвращает тот же класс
        self.name = name
        return self

class AdvancedBuilder(Builder):
    def set_level(self, level: int) -> Self:
        self.level = level
        return self

# -> Self лучше чем -> "Builder":
# AdvancedBuilder().set_name('x') возвращает AdvancedBuilder, не Builder
b: AdvancedBuilder = AdvancedBuilder().set_name('x').set_level(2)  # OK
```

---

## ParamSpec и Concatenate

Для типизации декораторов, которые сохраняют сигнатуру исходной функции:

```python
from typing import ParamSpec, Callable, TypeVar

P = ParamSpec('P')   # параметры функции
R = TypeVar('R')     # возвращаемый тип

# Callable[[int, str], bool] теряет имена параметров.
# ParamSpec позволяет "захватить" полную сигнатуру:

def log(func: Callable[P, R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f'Calling {func.__name__}')
        return func(*args, **kwargs)
    return wrapper

@log
def add(a: int, b: int) -> int:
    return a + b

add(1, 2)    # mypy знает: два int → int
add('x', 2)  # Error — правильно проверяет типы аргументов
```

### Concatenate — добавление аргумента в декораторе

```python
from typing import Concatenate

# Декоратор добавляет первый аргумент request: Request
def requires_auth(func: Callable[Concatenate[Request, P], R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        request = get_current_request()
        return func(request, *args, **kwargs)
    return wrapper
```

---

## TypeGuard

Сообщает mypy, что после вызова функции тип переменной уточнён:

```python
from typing import TypeGuard

def is_str_list(val: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(x, str) for x in val)

def process(items: list[object]) -> None:
    if is_str_list(items):
        # mypy знает: items: list[str]
        items[0].upper()   # OK
```

**TypeGuard vs isinstance:** `isinstance` работает с одним объектом, `TypeGuard` — для любой логики проверки, включая коллекции.

---

## Never и NoReturn

```python
from typing import Never, NoReturn  # Never — Python 3.11+

# NoReturn — функция никогда не возвращает (бросает исключение или бесконечный цикл)
def fail(msg: str) -> NoReturn:
    raise RuntimeError(msg)

# Never — тип, который не может существовать (нижний тип)
def assert_never(x: Never) -> Never:
    raise AssertionError(f'Unexpected value: {x}')

# Паттерн exhaustiveness check:
def handle(x: int | str) -> str:
    if isinstance(x, int):
        return str(x)
    elif isinstance(x, str):
        return x
    else:
        assert_never(x)  # mypy проверяет что все случаи обработаны

# NoReturn vs Never:
# NoReturn — возвращаемый тип функции
# Never   — тип значения (нижний тип, подтип всего)
```

---

## Annotated

Добавляет метаданные к аннотации — используется Pydantic, FastAPI и другими для валидации:

```python
from typing import Annotated
from pydantic import Field

# Annotated[type, *metadata] — type для mypy, metadata для runtime-инструментов
class User(BaseModel):
    name: Annotated[str, Field(min_length=1, max_length=100)]
    age:  Annotated[int, Field(ge=0, le=150)]

# FastAPI использует Annotated для зависимостей
from fastapi import Depends

def get_db() -> Session: ...

def endpoint(db: Annotated[Session, Depends(get_db)]) -> None: ...
```

---

## Forward references

Когда тип ещё не определён в момент аннотации:

```python
# Проблема: Node ещё не определён
class Node:
    def __init__(self, next: Node) -> None:  # NameError!
        self.next = next

# Решение 1: строка
class Node:
    def __init__(self, next: 'Node') -> None:
        self.next = next

# Решение 2: from __future__ import annotations (делает ВСЕ аннотации lazy)
from __future__ import annotations

class Node:
    def __init__(self, next: Node) -> None:  # OK
        self.next = next
```

`from __future__ import annotations` — все аннотации становятся строками и вычисляются лениво. Может сломать код, который читает аннотации в рантайме через `__annotations__` напрямую.

---

## Runtime typing

Аннотации хранятся в `__annotations__` как строки (если `from __future__ import annotations`) или как объекты типов.

```python
import typing

def greet(name: str) -> str: ...

greet.__annotations__  # {'name': str, 'return': str}

# Получить с разыменованием строковых аннотаций
typing.get_type_hints(greet)  # {'name': <class 'str'>, 'return': <class 'str'>}

# Проверка типа в runtime — НЕ используйте isinstance с дженериками
isinstance([1, 2], list[int])   # TypeError — нельзя так!
isinstance([1, 2], list)        # True — только базовый тип
```

---

## Инструменты

| Инструмент | Описание |
|---|---|
| **mypy** | Оригинальный type checker, `mypy --strict` для строгого режима |
| **Pyright** | Быстрый checker от Microsoft, используется в Pylance (VSCode) |
| **Ruff** | Линтер + некоторые type checks |

```bash
mypy app.py
mypy --strict app.py        # включает все проверки
# type: ignore               # подавить конкретную ошибку
# type: ignore[attr-defined] # подавить конкретный код ошибки
```

---

---

# Вопросы

## 🟢 Базовые

**Q: Что такое type hints и зачем нужны?**
Аннотации типов для переменных, параметров и возвращаемых значений. Не влияют на работу программы — нужны для статических анализаторов, IDE и документации.

---

**Q: Проверяются ли типы во время выполнения?**
Нет. Python остаётся динамически типизированным. Аннотации игнорируются интерпретатором. Проверка — только статическими инструментами (mypy, Pyright) или runtime-библиотеками (Pydantic, beartype).

---

**Q: `list[int]` или `List[int]` — что лучше?**
`list[int]` (Python 3.9+) — предпочтительнее: не требует импорта, короче, это встроенный дженерик. `List[int]` из `typing` — для совместимости с Python < 3.9.

---

**Q: Чем отличаются `Any`, `object`, `type`?**

- `Any` — совместим с **любым** типом в обе стороны, отключает проверку. Используй для legacy-кода.
- `object` — базовый класс всего. Строго типизирован: принимает любой экземпляр, но доступны только методы `object`.
- `type` — принимает сам **класс** (не экземпляр). Используй для фабрик и метаклассов.

```python
x: Any    = 42;  x.foo()     # OK для mypy
x: object = 42;  x.foo()     # Error
x: type   = int; x()         # OK — int это класс
```

---

**Q: В чём разница `Any` и `object`?**
`Any` совместим в обе стороны: `Any` присваивается любому типу и любой тип присваивается `Any`. `object` — только в одну сторону: принять можно что угодно, но присвоить `object` в `int` нельзя. Ключевое: mypy не проверяет методы у `Any`, а у `object` — проверяет строго.

---

**Q: `Optional[int]` vs `int | None`?**
Семантически одинаковы. `Optional[int]` — старый синтаксис, `int | None` — современный (Python 3.10+). Предпочитайте `int | None` в новом коде.

---

**Q: Что такое `Union`? Когда использовать `|`?**
`Union[X, Y]` — значение может быть типа X или Y. Оператор `|` (3.10+) — синтаксический сахар: `X | Y == Union[X, Y]`. Используйте `|` в новом коде.

---

**Q: Что делает `Literal["GET", "POST"]`?**
Ограничивает допустимые значения конкретными литералами. Mypy выдаст ошибку, если передать значение вне указанного набора. Полезно для строковых флагов, режимов, статусов.

---

## 🟡 Generics

**Q: Что такое Generic?**
Механизм параметрического полиморфизма — класс или функция, которая работает с разными типами, сохраняя информацию о типе. Реализуется через `TypeVar` и `Generic[T]`.

---

**Q: Для чего нужен `TypeVar`?**
Переменная типа — позволяет выразить связь между типами аргументов и возвращаемым значением:

```python
T = TypeVar('T')
def identity(x: T) -> T: return x  # вернёт тот же тип, что получил
```

---

**Q: Что такое `TypeVar(bound=...)`?**
Ограничивает T классом или его подклассами:
```python
T = TypeVar('T', bound=int)  # T может быть int, bool, или подклассом int
```

---

**Q: Что такое constrained TypeVar?**
Ограничивает T конкретным перечислением типов (не подклассами):
```python
T = TypeVar('T', int, str)  # только int или str, не их подклассы
```
**bound vs constrained:** `bound=X` — X и любые подклассы. `TypeVar('T', X, Y)` — строго X или Y.

---

**Q: Как типизировать Generic-контейнер?**
```python
class Stack(Generic[T]):
    def push(self, item: T) -> None: ...
    def pop(self) -> T: ...
```

---

## 🟡 Collections

**Q: Чем отличаются `Iterable`, `Iterator`, `Generator`?**

- `Iterable` — есть `__iter__`, может возвращать итераторы многократно (list, str)
- `Iterator` — есть `__iter__` + `__next__`, одноразовый, помнит позицию
- `Generator` — Iterator + `send`, `throw`, `close` (функция с yield)

```python
# Iterable но не Iterator
lst = [1, 2, 3]
# Iterator (одноразовый)
it = iter(lst)
```

---

**Q: Что возвращает `iter()`?**
`Iterator` — объект с `__next__`. Вызывает `__iter__()` у объекта. Сам итератор тоже является `Iterable` (возвращает `self` из `__iter__`).

---

**Q: Как типизировать генератор? Что означают три параметра?**
```python
Generator[YieldType, SendType, ReturnType]
Generator[int, None, str]
#          |     |     └── тип return из функции-генератора
#          |     └──────── тип, принимаемый через .send()
#          └────────────── тип, выдаваемый yield
```
Если генератор только yield и не использует send/return: `Iterator[int]`.

---

**Q: Разница `Sequence` и `Collection`?**

- `Sequence` — упорядочена, поддерживает индексацию (`__getitem__`), имеет длину
- `Collection` — только `__len__` + `__iter__` + `__contains__`, без индексации

---

## 🟠 Callable

**Q: Как типизировать функцию?**
```python
Callable[[int, str], bool]  # принимает (int, str), возвращает bool
Callable[[], None]           # без аргументов, возвращает None
Callable[..., Any]           # любые аргументы, любой возвращаемый тип
```

**Q: Что означает `Callable[..., Any]`?**
Функция с произвольными аргументами и произвольным возвращаемым типом. `...` (Ellipsis) означает "любая сигнатура".

---

## 🔴 Protocol

**Q: Что такое Protocol и чем отличается от ABC?**

| | Protocol | ABC |
|---|---|---|
| Типизация | Структурная | Номинальная |
| Наследование | Не нужно | Обязательно |
| Когда | Сторонние классы | Контролируешь иерархию |

---

**Q: Что такое structural typing?**
Тип определяется набором методов/атрибутов, а не иерархией наследования. "Если ходит как утка — значит утка." Protocol реализует structural typing в системе типов Python.

---

**Q: Когда лучше использовать Protocol?**
Когда не контролируешь классы (сторонние библиотеки), или когда явное наследование было бы излишним. Protocol + structural typing = более гибкий, слабосвязанный код.

---

## 🔴 TypedDict

**Q: Что такое TypedDict?**
Словарь с типизированными ключами. В runtime — обычный `dict`, но mypy проверяет ключи и типы значений.

**Q: TypedDict vs dataclass vs dict?**

| | `dict` | `TypedDict` | `dataclass` |
|---|---|---|---|
| Тип | dict | dict | class |
| Проверка | нет | статическая | статическая |
| Методы | нет | нет | да |
| Default | нет | нет (`total=False`) | да |
| JSON-совместим | да | да | нет |

**Q: `NotRequired` и `Required`?**
```python
class Config(TypedDict, total=False):  # все поля необязательны
    host: Required[str]               # но это — обязательное
    port: NotRequired[int]            # явно необязательное
```

---

## 🔴 NewType

**Q: Для чего нужен `NewType`?**
Создаёт различимый тип-обёртку. В runtime это identity-функция (нет overhead), но mypy считает его **отдельным типом**:

```python
UserId = NewType('UserId', int)
PostId = NewType('PostId', int)

get_user(UserId(1))  # OK
get_user(PostId(1))  # Error — предотвращает путаницу ID
get_user(1)          # Error — int != UserId
```

---

## 🔴 Self

**Q: Что делает `Self` и почему лучше чем `-> "MyClass"`?**
При наследовании `-> Self` возвращает тип **фактического класса**, а не родителя:

```python
class Builder:
    def configure(self) -> Self: ...  # возвращает AdvancedBuilder при вызове на нём

# -> "Builder" вернёт Builder, даже если вызван на AdvancedBuilder
# -> Self вернёт правильный подкласс
```

---

## 🔴 ParamSpec

**Q: Что такое ParamSpec и как типизировать декоратор?**
`ParamSpec` захватывает полную сигнатуру функции (имена параметров, их типы), в отличие от `Callable[..., R]` который теряет сигнатуру:

```python
P = ParamSpec('P')
R = TypeVar('R')

def log(func: Callable[P, R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        return func(*args, **kwargs)
    return wrapper
```

**Q: Почему обычный `Callable` не подходит для декоратора?**
`Callable[..., R]` теряет информацию об аргументах — mypy не может проверить правильность вызова задекорированной функции. `ParamSpec` сохраняет полную сигнатуру.

---

## 🔴 TypeGuard

**Q: Что такое TypeGuard?**
Сообщает mypy что после вызова этой функции тип аргумента уточнён:

```python
def is_list_of_str(val: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(x, str) for x in val)

if is_list_of_str(items):
    items[0].upper()  # mypy знает: list[str]
```

**TypeGuard vs isinstance:** `isinstance` — для одного объекта, встроен в mypy. `TypeGuard` — для произвольной логики проверки.

---

## 🔴 Never

**Q: `Never` vs `NoReturn`?**

- `NoReturn` — тип функции, которая **никогда не возвращает** (бросает исключение, бесконечный цикл)
- `Never` — тип значения, которое **не может существовать** (нижний тип, подтип всех типов)

```python
def crash() -> NoReturn: raise RuntimeError()
def assert_never(x: Never) -> Never: raise AssertionError()

# Паттерн exhaustive check:
match x:
    case int(): ...
    case str(): ...
    case _: assert_never(x)  # mypy проверит что все случаи закрыты
```

---

## 🔴 Annotated

**Q: Что делает `Annotated`?**
Добавляет метаданные к типу. Mypy видит только первый аргумент (тип), runtime-инструменты (Pydantic, FastAPI) читают метаданные:

```python
Annotated[int, Field(ge=0, le=100)]  # mypy: int, Pydantic: 0..100
Annotated[str, Field(max_length=50)]
```

---

## 🔴 Forward references

**Q: Почему пишут `def foo(x: "MyClass")`?**
Если тип ещё не определён в момент аннотации (например, класс ссылается на себя или на класс ниже). Строка — это forward reference, вычисляется лениво.

**Q: Что делает `from __future__ import annotations`?**
Делает все аннотации в модуле lazy (строками). Позволяет использовать типы до их объявления без кавычек. Может сломать код, который читает `__annotations__` напрямую в рантайме.

---

## 🔴 Практические задачи

**Q: Типизировать `first(items) → items[0]`:**
```python
T = TypeVar('T')
def first(items: Sequence[T]) -> T:
    return items[0]
```

**Q: Типизировать декоратор:**
```python
P = ParamSpec('P')
R = TypeVar('R')

def cache(func: Callable[P, R]) -> Callable[P, R]:
    memo: dict[Any, R] = {}
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        key = (args, tuple(sorted(kwargs.items())))
        if key not in memo:
            memo[key] = func(*args, **kwargs)
        return memo[key]
    return wrapper
```

**Q: Generic Repository:**
```python
T = TypeVar('T')

class Repository(Generic[T]):
    def get(self, id: int) -> T | None: ...
    def save(self, entity: T) -> None: ...
    def list(self) -> list[T]: ...

class UserRepository(Repository[User]): ...
```

**Q: Типизировать flatten:**
```python
from typing import Any

def flatten(data: list[Any]) -> list[Any]:
    result = []
    for item in data:
        if isinstance(item, list):
            result.extend(flatten(item))
        else:
            result.append(item)
    return result
```
