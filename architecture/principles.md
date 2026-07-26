# Архитектура — Принципы проектирования

## SOLID

Пять принципов объектно-ориентированного проектирования.

### S — Single Responsibility Principle (Единственная ответственность)

Каждый класс должен иметь одну причину для изменения.

```python
# Плохо: класс делает и бизнес-логику, и I/O, и уведомления
class Order:
    def calculate_total(self): ...
    def save_to_db(self): ...
    def send_confirmation_email(self): ...

# Хорошо: разделить по ответственностям
class Order:
    def calculate_total(self): ...

class OrderRepository:
    def save(self, order: Order): ...

class OrderNotifier:
    def send_confirmation(self, order: Order): ...
```

### O — Open/Closed Principle (Открытость-закрытость)

Открыт для расширения, закрыт для изменения. Новое поведение добавляется через наследование/композицию без изменения существующего кода.

```python
# Плохо: добавление нового типа требует изменения функции
def calculate_area(shape):
    if shape.type == 'circle': ...
    elif shape.type == 'square': ...   # +1 тип = изменение кода

# Хорошо: каждая фигура сама знает свою площадь
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Circle(Shape):
    def area(self) -> float: return math.pi * self.r ** 2

class Triangle(Shape):              # новый тип — без изменений выше
    def area(self) -> float: ...
```

### L — Liskov Substitution Principle (Подстановка Лисков)

Объект дочернего класса можно подставить вместо родительского без нарушения работы программы.

```python
# Нарушение: Square меняет контракт Rectangle
class Rectangle:
    def set_width(self, w): self.width = w
    def set_height(self, h): self.height = h

class Square(Rectangle):
    def set_width(self, w):
        self.width = self.height = w   # нарушает ожидания Rectangle!

# Код, работающий с Rectangle, сломается при подстановке Square
def make_wider(rect: Rectangle):
    old_h = rect.height
    rect.set_width(rect.width + 1)
    assert rect.height == old_h   # упадёт для Square
```

### I — Interface Segregation Principle (Разделение интерфейса)

Клиенты не должны зависеть от методов, которые не используют. Один большой интерфейс → несколько маленьких.

```python
# Плохо: Dog вынужден реализовывать fly()
class Animal(ABC):
    def fly(self): ...
    def swim(self): ...
    def run(self): ...

# Хорошо: отдельные интерфейсы
class Flyable(ABC):
    @abstractmethod
    def fly(self): ...

class Swimmable(ABC):
    @abstractmethod
    def swim(self): ...

class Duck(Flyable, Swimmable): ...     # утка умеет оба
class Penguin(Swimmable): ...          # пингвин только плавает
class Eagle(Flyable): ...              # орёл только летает
```

### D — Dependency Inversion Principle (Инверсия зависимостей)

Модули верхнего уровня не зависят от нижнего. Оба зависят от абстракций.

```python
# Плохо: высокоуровневый класс зависит от конкретной реализации
class OrderService:
    def __init__(self):
        self.repo = PostgresOrderRepository()   # жёсткая привязка к Postgres

# Хорошо: зависимость от абстракции
class OrderRepository(ABC):
    @abstractmethod
    def save(self, order: Order): ...

class OrderService:
    def __init__(self, repo: OrderRepository):  # принимает любую реализацию
        self.repo = repo

# В тестах: подменяем реализацию
service = OrderService(repo=InMemoryOrderRepository())
```

---

## DRY, YAGNI, KISS

**DRY (Don't Repeat Yourself)** — каждая единица знания должна существовать в одном месте. Дублирование = дублирование поддержки и возможность рассинхронизации.

**YAGNI (You Aren't Gonna Need It)** — не пиши код который сейчас не нужен. Не проектируй «на будущее» без конкретной потребности. При рефакторинге — смело удаляй неиспользуемое.

**KISS (Keep It Simple, Stupid)** — простое решение предпочтительнее сложного. Простой код легче читать, тестировать и менять.

---

## Принципы ООП

### Инкапсуляция

Объединение данных и методов в единый объект. Скрытие деталей реализации.

**Важно:** инкапсуляция ≠ сокрытие данных. Главная цель — собрать знания об объекте в одном месте. Сокрытие — лишь один из инструментов.

```python
class Money:
    def __init__(self, amount: Decimal, currency: str):
        self._amount = amount      # детали хранения скрыты
        self._currency = currency

    def add(self, other: 'Money') -> 'Money':
        if self._currency != other._currency:
            raise ValueError("Different currencies")
        return Money(self._amount + other._amount, self._currency)
    # Пользователь не думает о float precision, округлении и т.д.
```

### Наследование

Создание нового класса на основе существующего. В «чистом» ООП нужно для полиморфизма. **Как самостоятельный инструмент — предпочитай композицию**: наследование создаёт сильное связывание.

### Полиморфизм

Одно имя — разные реализации в зависимости от типа объекта.

- **Подтипов** — переопределение методов в наследнике (классический ООП)
- **Параметрический** — Generic/шаблоны, один алгоритм для разных типов
- **Ad hoc (специальный)** — перегрузка функций

```python
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

# Полиморфный вызов — не знаем конкретный тип
for shape in [Circle(r=5), Square(side=4), Triangle(...)]:
    print(shape.area())   # каждый считает по-своему
```

### Абстракция

Выделение существенных характеристик, скрытие несущественных деталей. Позволяет работать с концепцией, не зная реализации.

```python
# Нам не важно, как именно отправляется письмо — SMTP, SES, API
class EmailSender(ABC):
    @abstractmethod
    def send(self, to: str, subject: str, body: str) -> None: ...
```

---

## Связность и Сцепленность

**Coupling (Сцепленность между модулями)** — степень зависимости модулей друг от друга.
- **Низкое** (loose coupling) — хорошо: модули можно менять независимо
- **Высокое** (tight coupling) — плохо: изменение одного ломает другие

**Cohesion (Связность внутри модуля)** — насколько элементы модуля служат одной цели.
- **Высокая** (high cohesion) — хорошо: всё в классе про одно
- **Низкая** (low cohesion) — плохо: «класс-помойка»

**Цель:** **low coupling + high cohesion**.
