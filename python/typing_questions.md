# Python — Type Hints: Вопросы

> Теория: [typing.md](typing.md)

---

## Базовые

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

## Generics

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

## Collections

**Q: Чем отличаются `Iterable`, `Iterator`, `Generator`?**

- `Iterable` — есть `__iter__`, может возвращать итераторы многократно (list, str)
- `Iterator` — есть `__iter__` + `__next__`, одноразовый, помнит позицию
- `Generator` — Iterator + `send`, `throw`, `close` (функция с yield)

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

## Callable

**Q: Как типизировать функцию?**

```python
Callable[[int, str], bool]  # принимает (int, str), возвращает bool
Callable[[], None]           # без аргументов, возвращает None
Callable[..., Any]           # любые аргументы, любой возвращаемый тип
```

---

**Q: Что означает `Callable[..., Any]`?**

Функция с произвольными аргументами и произвольным возвращаемым типом. `...` (Ellipsis) означает "любая сигнатура".

---

## Protocol

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

## TypedDict

**Q: Что такое TypedDict?**

Словарь с типизированными ключами. В runtime — обычный `dict`, но mypy проверяет ключи и типы значений.

---

**Q: TypedDict vs dataclass vs dict?**

| | `dict` | `TypedDict` | `dataclass` |
|---|---|---|---|
| Тип | dict | dict | class |
| Проверка | нет | статическая | статическая |
| Методы | нет | нет | да |
| Default | нет | нет (`total=False`) | да |
| JSON-совместим | да | да | нет |

---

**Q: `NotRequired` и `Required`?**

```python
class Config(TypedDict, total=False):  # все поля необязательны
    host: Required[str]               # но это — обязательное
    port: NotRequired[int]            # явно необязательное
```

---

## NewType

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

## Self

**Q: Что делает `Self` и почему лучше чем `-> "MyClass"`?**

При наследовании `-> Self` возвращает тип **фактического класса**, а не родителя:

```python
class Builder:
    def configure(self) -> Self: ...  # возвращает AdvancedBuilder при вызове на нём

# -> "Builder" вернёт Builder, даже если вызван на AdvancedBuilder
# -> Self вернёт правильный подкласс
```

---

## ParamSpec

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

---

**Q: Почему обычный `Callable` не подходит для декоратора?**

`Callable[..., R]` теряет информацию об аргументах — mypy не может проверить правильность вызова задекорированной функции. `ParamSpec` сохраняет полную сигнатуру.

---

## TypeGuard

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

## Never

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

## Annotated

**Q: Что делает `Annotated`?**

Добавляет метаданные к типу. Mypy видит только первый аргумент (тип), runtime-инструменты (Pydantic, FastAPI) читают метаданные:

```python
Annotated[int, Field(ge=0, le=100)]  # mypy: int, Pydantic: 0..100
Annotated[str, Field(max_length=50)]
```

---

## Forward references

**Q: Почему пишут `def foo(x: "MyClass")`?**

Если тип ещё не определён в момент аннотации (например, класс ссылается на себя или на класс ниже). Строка — это forward reference, вычисляется лениво.

---

**Q: Что делает `from __future__ import annotations`?**

Делает все аннотации в модуле lazy (строками). Позволяет использовать типы до их объявления без кавычек. Может сломать код, который читает `__annotations__` напрямую в рантайме.

---

## Практические задачи

**Q: Типизировать `first(items) → items[0]`:**

```python
T = TypeVar('T')
def first(items: Sequence[T]) -> T:
    return items[0]
```

---

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

---

**Q: Generic Repository:**

```python
T = TypeVar('T')

class Repository(Generic[T]):
    def get(self, id: int) -> T | None: ...
    def save(self, entity: T) -> None: ...
    def list(self) -> list[T]: ...

class UserRepository(Repository[User]): ...
```

---

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
