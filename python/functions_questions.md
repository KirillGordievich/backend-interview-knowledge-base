# Python — Функции: Вопросы

> Теория: [functions.md](functions.md)

---

**Q: Какой порядок параметров в определении функции?**

```python
def f(pos_only, /, regular, *, kw_only, **kwargs):
```

1. Positional-only (до `/`)
2. Обычные (positional or keyword)
3. `*args` — сборщик позиционных
4. Keyword-only (после `*` или `*args`)
5. `**kwargs` — сборщик именованных

---

**Q: Что такое positional-only и keyword-only параметры?**

- **Positional-only** (до `/`) — можно передать только по позиции, не по имени
- **Keyword-only** (после `*`) — можно передать только по имени

```python
def f(a, /, b, *, c):
    pass

f(1, 2, c=3)   # OK
f(a=1, 2, c=3) # Error — a positional-only
f(1, 2, 3)     # Error — c keyword-only
```

---

**Q: Почему опасно использовать mutable default аргумент?**

Значение по умолчанию создаётся один раз при определении функции, а не при каждом вызове. Если это mutable объект — он разделяется между вызовами:

```python
def bad(items=[]):
    items.append(1)
    return items

bad()  # [1]
bad()  # [1, 1] — тот же список!

# Правильно:
def good(items=None):
    items = items or []
    items.append(1)
    return items
```

---

**Q: Что такое замыкание (closure)?**

Функция, которая захватывает переменные из внешней области видимости. Внутренняя функция "помнит" окружение, в котором была создана:

```python
def make_counter():
    count = 0
    def counter():
        nonlocal count
        count += 1
        return count
    return counter

c = make_counter()
c()  # 1
c()  # 2
```

---

**Q: Что такое `nonlocal` и когда нужен?**

`nonlocal` позволяет изменять переменную из внешней (но не глобальной) области видимости. Без `nonlocal` присваивание создаст локальную переменную вместо изменения внешней.

---

**Q: Как работают декораторы?**

Декоратор — функция, которая принимает функцию и возвращает новую (обычно обёртку). `@decorator` — синтаксический сахар для `func = decorator(func)`:

```python
def log(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print(f'Calling {func.__name__}')
        return func(*args, **kwargs)
    return wrapper

@log
def add(a, b):
    return a + b
```

---

**Q: Зачем нужен `functools.wraps`?**

Сохраняет метаданные оригинальной функции (`__name__`, `__doc__`, `__module__`). Без него задекорированная функция будет иметь имя и docstring обёртки, а не оригинала.

---

**Q: Как написать декоратор с аргументами?**

Нужен дополнительный уровень вложенности — фабрика декораторов:

```python
def repeat(n):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(n):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)  # repeat(3) возвращает decorator
def greet():
    print('Hello')
```

---

**Q: Что такое `lambda`? Когда использовать?**

Анонимная функция из одного выражения: `lambda x, y: x + y`. Используй для коротких callback'ов (`sorted(key=lambda x: x.name)`). Не используй для сложной логики — определи обычную функцию.

---

**Q: Что делает `functools.lru_cache`?**

Кэширует результаты функции по аргументам (мемоизация). Работает только с hashable аргументами:

```python
@functools.lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2: return n
    return fibonacci(n-1) + fibonacci(n-2)
```

---

**Q: Чем `lru_cache` отличается от `cache`?**

`@cache` (Python 3.9+) — это `lru_cache(maxsize=None)` — кэш без ограничения размера. `lru_cache` с `maxsize` — LRU-стратегия, вытесняет старые записи.

---

**Q: Что такое `functools.partial`?**

Фиксирует часть аргументов функции, возвращая новую функцию:

```python
from functools import partial

def power(base, exp):
    return base ** exp

square = partial(power, exp=2)
square(5)  # 25
```

---

**Q: Что такое LEGB правило?**

Порядок поиска переменных в Python:
- **L**ocal — локальная область функции
- **E**nclosing — области внешних функций (замыкание)
- **G**lobal — уровень модуля
- **B**uilt-in — встроенные имена (`print`, `len`)

---

**Q: Функции — это объекты первого класса. Что это значит?**

Функции можно: присваивать переменным, передавать как аргументы, возвращать из других функций, хранить в коллекциях. У них есть атрибуты (`__name__`, `__doc__`, `__defaults__`).
