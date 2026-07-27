# Паттерны проектирования (GoF)

23 классических паттерна из книги "Gang of Four". Делятся на три группы.

---

## Порождающие (Creational)

Описывают, **как создавать объекты**.

### Abstract Factory (Абстрактная фабрика)

Создаёт семейства взаимосвязанных объектов без указания конкретных классов.

```python
class WidgetFactory:
    def create_button(self): ...
    def create_scrollbar(self): ...

class MacFactory(WidgetFactory):
    def create_button(self): return MacButton()
    def create_scrollbar(self): return MacScrollbar()

class WinFactory(WidgetFactory):
    def create_button(self): return WinButton()
    def create_scrollbar(self): return WinScrollbar()

def create_dialog(factory: WidgetFactory):
    btn = factory.create_button()   # не знаем конкретный класс
```

**Когда:** нужно гарантировать совместимость создаваемых объектов (Mac-кнопки + Mac-скроллбары, не Win).

---

### Builder (Строитель)

Отделяет конструирование сложного объекта от его представления. Пошаговое создание.

```python
class QueryBuilder:
    def __init__(self):
        self._table = None
        self._conditions = []
        self._limit = None

    def from_table(self, table: str) -> 'QueryBuilder':
        self._table = table
        return self

    def where(self, condition: str) -> 'QueryBuilder':
        self._conditions.append(condition)
        return self

    def limit(self, n: int) -> 'QueryBuilder':
        self._limit = n
        return self

    def build(self) -> str:
        sql = f"SELECT * FROM {self._table}"
        if self._conditions:
            sql += " WHERE " + " AND ".join(self._conditions)
        if self._limit:
            sql += f" LIMIT {self._limit}"
        return sql

query = (QueryBuilder()
    .from_table("users")
    .where("age > 18")
    .limit(10)
    .build())
```

**Отличие от Фабрики:** строитель создаёт объект пошагово и возвращает его в конце; фабрика возвращает сразу.

---

### Factory Method (Фабричный метод)

Определяет интерфейс создания объекта, но позволяет подклассам решать, какой класс инстанциировать.

```python
class Localizer:
    def localize(self, msg: str) -> str: ...

class EnglishLocalizer(Localizer):
    def localize(self, msg): return msg

class GreekLocalizer(Localizer):
    translations = {"dog": "σκύλος"}
    def localize(self, msg): return self.translations.get(msg, msg)

def get_localizer(language: str = "English") -> Localizer:
    """Фабричная функция — клиент не знает о конкретных классах"""
    localizers = {"English": EnglishLocalizer, "Greek": GreekLocalizer}
    return localizers[language]()
```

---

### Prototype (Прототип)

Создаёт новые объекты клонированием существующего экземпляра.

```python
import copy

class Config:
    def __init__(self, settings: dict):
        self.settings = settings

    def clone(self) -> 'Config':
        return copy.deepcopy(self)

base = Config({"debug": False, "db": "postgres://prod"})
dev = base.clone()
dev.settings["debug"] = True   # base не затронут
```

Полезен когда создание объекта дорого, а клонирование — дёшево.

---

### Singleton (Одиночка)

Гарантирует единственный экземпляр класса.

```python
class DatabasePool:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, url: str):
        if not hasattr(self, '_initialized'):
            self._pool = create_pool(url)
            self._initialized = True

# Простейший способ в Python — модуль с глобальным состоянием
# config.py
_config = {}

def get_config():
    if not _config:
        _config.update(load_from_env())
    return _config
```

**Проблемы:** затрудняет тестирование, скрытые зависимости. В Python предпочтительнее DI.

---

## Структурные (Structural)

Описывают, **как из объектов составлять более крупные структуры**.

### Adapter (Адаптер)

Обеспечивает совместимость несовместимых интерфейсов.

```python
# Устаревший класс с неудобным интерфейсом
class OldPaymentSystem:
    def pay_in_cents(self, cents: int): ...

# Целевой интерфейс
class PaymentGateway:
    def pay(self, amount: float): ...

# Адаптер — оборачивает старый класс, предоставляет новый интерфейс
class PaymentAdapter(PaymentGateway):
    def __init__(self, old: OldPaymentSystem):
        self._old = old

    def pay(self, amount: float):
        self._old.pay_in_cents(int(amount * 100))
```

**Отличие от Фасада:** Адаптер приводит интерфейс к нужному (1-к-1), Фасад упрощает сложную систему.

---

### Bridge (Мост)

Разделяет абстракцию и реализацию, позволяя изменять их независимо.

```python
# Реализации (независимая иерархия)
class Renderer:
    def render_circle(self, x, y, radius): ...

class SVGRenderer(Renderer):
    def render_circle(self, x, y, radius):
        print(f'<circle cx="{x}" cy="{y}" r="{radius}"/>')

class CanvasRenderer(Renderer):
    def render_circle(self, x, y, radius):
        print(f'ctx.arc({x}, {y}, {radius})')

# Абстракции (своя иерархия)
class Circle:
    def __init__(self, x, y, radius, renderer: Renderer):
        self.x, self.y, self.radius = x, y, radius
        self.renderer = renderer   # ← мост

    def draw(self):
        self.renderer.render_circle(self.x, self.y, self.radius)
```

---

### Composite (Компоновщик)

Позволяет единообразно обрабатывать отдельные объекты и их составные группы (дерево).

```python
from abc import ABC, abstractmethod

class FileSystemItem(ABC):
    @abstractmethod
    def size(self) -> int: ...

class File(FileSystemItem):
    def __init__(self, name: str, size: int):
        self._size = size
    def size(self): return self._size

class Directory(FileSystemItem):
    def __init__(self):
        self._children: list[FileSystemItem] = []
    def add(self, item: FileSystemItem):
        self._children.append(item)
    def size(self):
        return sum(child.size() for child in self._children)

root = Directory()
root.add(File("readme.txt", 100))
sub = Directory()
sub.add(File("main.py", 500))
root.add(sub)
root.size()  # 600 — одинаково для File и Directory
```

---

### Decorator (Декоратор)

Динамически добавляет объекту новые обязанности без изменения его класса.

```python
from functools import wraps
import time

# Функциональный декоратор Python
def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__}: {time.perf_counter() - start:.3f}s")
        return result
    return wrapper

# Классический ОО-декоратор (оборачивает объект)
class LoggedRepository:
    def __init__(self, repo):
        self._repo = repo

    def find(self, id):
        print(f"find({id})")
        result = self._repo.find(id)
        print(f"→ {result}")
        return result
```

---

### Facade (Фасад)

Предоставляет простой интерфейс к сложной подсистеме.

```python
# Фасад скрывает несколько сложных подсистем
class MediaPlayer:
    def __init__(self):
        self._video = VideoDecoder()
        self._audio = AudioDecoder()
        self._subs = SubtitleParser()
        self._renderer = VideoRenderer()

    def play(self, filepath: str):
        video = self._video.decode(filepath)
        audio = self._audio.decode(filepath)
        subs = self._subs.parse(filepath)
        self._renderer.render(video, audio, subs)

# Клиент не знает о внутренних подсистемах
MediaPlayer().play("movie.mkv")
```

---

### Flyweight (Приспособленец)

Минимизирует использование памяти, разделяя общее состояние между множеством мелких объектов.

```python
class Glyph:
    """Символ шрифта — один объект на каждый уникальный (char, font)"""
    _pool: dict = {}

    def __new__(cls, char: str, font: str):
        key = (char, font)
        if key not in cls._pool:
            obj = super().__new__(cls)
            obj.char = char
            obj.font = font
            cls._pool[key] = obj
        return cls._pool[key]

g1 = Glyph("A", "Arial")
g2 = Glyph("A", "Arial")
assert g1 is g2   # один и тот же объект
```

Python реализует Flyweight неявно: интернирование строк (`sys.intern`), пул малых целых чисел (-5..256).

---

### Proxy (Заместитель)

Подставляет объект-суррогат, контролирующий доступ к реальному объекту.

```python
class ImageProxy:
    """Виртуальный прокси — ленивая загрузка"""
    def __init__(self, filepath: str):
        self._filepath = filepath
        self._image = None

    def display(self):
        if self._image is None:
            self._image = RealImage(self._filepath)  # загружаем только по требованию
        self._image.display()

# Виды заместителей:
# - Виртуальный (lazy loading)
# - Защитный (проверка прав)
# - Удалённый (прокси к remote-объекту)
# - Кэширующий
```

---

## Поведенческие (Behavioral)

Описывают **алгоритмы и взаимодействие объектов**.

### Chain of Responsibility (Цепочка ответственности)

Передаёт запрос по цепочке обработчиков; каждый решает: обработать или передать дальше.

```python
from abc import ABC, abstractmethod

class Handler(ABC):
    def __init__(self, successor=None):
        self._next = successor

    def handle(self, request):
        if self._next:
            return self._next.handle(request)

class AuthHandler(Handler):
    def handle(self, request):
        if not request.get("token"):
            return "401 Unauthorized"
        return super().handle(request)

class RateLimitHandler(Handler):
    def handle(self, request):
        if request.get("rate_exceeded"):
            return "429 Too Many Requests"
        return super().handle(request)

class BusinessHandler(Handler):
    def handle(self, request):
        return f"200 OK: {request['data']}"

handler = AuthHandler(RateLimitHandler(BusinessHandler()))
handler.handle({"token": "abc", "data": "payload"})  # 200 OK
handler.handle({})                                     # 401 Unauthorized
```

---

### Command (Команда)

Инкапсулирует запрос как объект. Поддерживает отмену, логирование, очереди.

```python
from abc import ABC, abstractmethod

class Command(ABC):
    @abstractmethod
    def execute(self): ...
    @abstractmethod
    def undo(self): ...

class MoveCommand(Command):
    def __init__(self, piece, dx, dy):
        self.piece = piece
        self.dx, self.dy = dx, dy

    def execute(self): self.piece.move(self.dx, self.dy)
    def undo(self):    self.piece.move(-self.dx, -self.dy)

class CommandHistory:
    def __init__(self):
        self._history: list[Command] = []

    def execute(self, cmd: Command):
        cmd.execute()
        self._history.append(cmd)

    def undo(self):
        if self._history:
            self._history.pop().undo()
```

---

### Iterator (Итератор)

Предоставляет способ последовательного обхода коллекции без раскрытия её структуры.

```python
# В Python встроено: __iter__ + __next__
class NumberRange:
    def __init__(self, start, stop, step=1):
        self.start, self.stop, self.step = start, stop, step

    def __iter__(self):
        current = self.start
        while current < self.stop:
            yield current
            current += self.step

# Генератор — краткая форма итератора
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b
```

---

### Mediator (Посредник)

Определяет объект, инкапсулирующий взаимодействие множества объектов. Снижает связность.

```python
class ChatRoom:
    """Посредник — все общаются только через него"""
    def __init__(self):
        self._users: list['User'] = []

    def join(self, user: 'User'):
        self._users.append(user)
        user.room = self

    def send(self, message: str, sender: 'User'):
        for user in self._users:
            if user is not sender:
                user.receive(message, sender.name)

class User:
    def __init__(self, name: str):
        self.name = name
        self.room: ChatRoom | None = None

    def send(self, message: str):
        self.room.send(message, self)  # через посредника, не напрямую

    def receive(self, message: str, from_: str):
        print(f"[{self.name}] {from_}: {message}")
```

---

### Memento (Хранитель)

Сохраняет и восстанавливает внутреннее состояние без нарушения инкапсуляции.

```python
from dataclasses import dataclass

@dataclass
class EditorState:
    content: str
    cursor: int

class TextEditor:
    def __init__(self):
        self.content = ""
        self.cursor = 0
        self._history: list[EditorState] = []

    def type(self, text: str):
        self._history.append(EditorState(self.content, self.cursor))
        self.content += text
        self.cursor += len(text)

    def undo(self):
        if self._history:
            state = self._history.pop()
            self.content = state.content
            self.cursor = state.cursor
```

---

### Observer (Наблюдатель)

Определяет зависимость «один-ко-многим»: при изменении объекта все подписчики автоматически уведомляются.

```python
from collections import defaultdict
from typing import Callable

class EventBus:
    def __init__(self):
        self._listeners: dict[str, list[Callable]] = defaultdict(list)

    def subscribe(self, event: str, handler: Callable):
        self._listeners[event].append(handler)

    def publish(self, event: str, data=None):
        for handler in self._listeners[event]:
            handler(data)

bus = EventBus()
bus.subscribe("user.created", lambda u: send_welcome_email(u))
bus.subscribe("user.created", lambda u: create_default_settings(u))
bus.publish("user.created", {"email": "alice@example.com"})
```

---

### State (Состояние)

Позволяет объекту менять поведение при изменении внутреннего состояния.

```python
from abc import ABC, abstractmethod

class OrderState(ABC):
    @abstractmethod
    def confirm(self, order): ...
    @abstractmethod
    def cancel(self, order): ...

class PendingState(OrderState):
    def confirm(self, order): order.state = ConfirmedState()
    def cancel(self, order):  order.state = CancelledState()

class ConfirmedState(OrderState):
    def confirm(self, order): raise ValueError("Already confirmed")
    def cancel(self, order):  order.state = CancelledState()

class CancelledState(OrderState):
    def confirm(self, order): raise ValueError("Cannot confirm cancelled")
    def cancel(self, order):  raise ValueError("Already cancelled")

class Order:
    def __init__(self):
        self.state: OrderState = PendingState()

    def confirm(self): self.state.confirm(self)
    def cancel(self):  self.state.cancel(self)
```

---

### Strategy (Стратегия)

Определяет семейство алгоритмов, инкапсулирует их и делает взаимозаменяемыми.

```python
from typing import Protocol

class SortStrategy(Protocol):
    def sort(self, data: list) -> list: ...

class QuickSort:
    def sort(self, data: list) -> list: ...

class MergeSort:
    def sort(self, data: list) -> list: ...

class Sorter:
    def __init__(self, strategy: SortStrategy):
        self._strategy = strategy

    def sort(self, data: list) -> list:
        return self._strategy.sort(data)

sorter = Sorter(QuickSort())
sorter.sort([3, 1, 2])
sorter._strategy = MergeSort()   # можно менять на лету
sorter.sort([3, 1, 2])
```

---

### Template Method (Шаблонный метод)

Определяет скелет алгоритма в базовом классе; подклассы переопределяют отдельные шаги.

```python
from abc import ABC, abstractmethod

class DataMigration(ABC):
    def run(self):              # шаблонный метод — неизменяемый скелет
        data = self.extract()
        transformed = self.transform(data)
        self.load(transformed)

    @abstractmethod
    def extract(self): ...

    def transform(self, data):  # шаг с реализацией по умолчанию
        return data

    @abstractmethod
    def load(self, data): ...

class CSVToPostgresMigration(DataMigration):
    def extract(self):
        return read_csv("data.csv")
    def transform(self, data):
        return [row.strip() for row in data]
    def load(self, data):
        db.bulk_insert(data)
```

---

### Visitor (Посетитель)

Позволяет добавлять операции к объектам без изменения их классов.

```python
from abc import ABC, abstractmethod

class ASTNode(ABC):
    @abstractmethod
    def accept(self, visitor): ...

class NumberNode(ASTNode):
    def __init__(self, value: float): self.value = value
    def accept(self, visitor): return visitor.visit_number(self)

class AddNode(ASTNode):
    def __init__(self, left, right): self.left, self.right = left, right
    def accept(self, visitor): return visitor.visit_add(self)

class Evaluator:
    def visit_number(self, node): return node.value
    def visit_add(self, node):
        return node.left.accept(self) + node.right.accept(self)

class Printer:
    def visit_number(self, node): return str(node.value)
    def visit_add(self, node):
        return f"({node.left.accept(self)} + {node.right.accept(self)})"

tree = AddNode(NumberNode(1), NumberNode(2))
tree.accept(Evaluator())  # 3.0
tree.accept(Printer())    # "(1.0 + 2.0)"
```

---

## Итоговая таблица

| Паттерн | Группа | Суть |
|---|---|---|
| Abstract Factory | Creational | Семейства совместимых объектов |
| Builder | Creational | Пошаговое создание |
| Factory Method | Creational | Подкласс выбирает конкретный класс |
| Prototype | Creational | Клонирование |
| Singleton | Creational | Один экземпляр |
| Adapter | Structural | Совместимость несовместимых интерфейсов |
| Bridge | Structural | Абстракция ⟂ реализация |
| Composite | Structural | Дерево объектов (часть/целое) |
| Decorator | Structural | Динамические обязанности |
| Facade | Structural | Простой интерфейс к сложной системе |
| Flyweight | Structural | Разделяемое состояние в памяти |
| Proxy | Structural | Контроль доступа к объекту |
| Chain of Responsibility | Behavioral | Цепочка обработчиков |
| Command | Behavioral | Запрос как объект (undo/redo) |
| Iterator | Behavioral | Обход коллекции |
| Mediator | Behavioral | Централизованное взаимодействие |
| Memento | Behavioral | Снапшот состояния |
| Observer | Behavioral | Подписка на события |
| State | Behavioral | Поведение зависит от состояния |
| Strategy | Behavioral | Взаимозаменяемые алгоритмы |
| Template Method | Behavioral | Скелет алгоритма в базовом классе |
| Visitor | Behavioral | Новые операции без изменения классов |
