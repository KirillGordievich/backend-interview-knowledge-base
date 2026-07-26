# Python — Функции

## *args и **kwargs

`*args` — кортеж позиционных аргументов, переданных сверх объявленных параметров.
`**kwargs` — словарь именованных аргументов, переданных сверх объявленных параметров.

```python
def foo(*args, **kwargs):
    print(args)    # кортеж, никогда не None
    print(kwargs)  # словарь, никогда не None

foo(1, 2, x=3)
# (1, 2)
# {'x': 3}

foo()
# ()
# {}
```

Звёздочки можно использовать и при вызове для распаковки:

```python
def add(a, b, c):
    return a + b + c

args = [1, 2, 3]
add(*args)          # 6

kwargs = {'a': 1, 'b': 2, 'c': 3}
add(**kwargs)       # 6
```

---

## Порядок параметров в сигнатуре

```python
def func(pos1, pos2, /, normal, *, kw_only, **kwargs):
    pass
#         ^--- только позиционные (Python 3.8+)
#                      ^--- позиционные или именованные
#                              ^--- только именованные
```

---

## Изменяемые объекты как параметры по умолчанию

**Проблема:** значения по умолчанию создаются **один раз** при определении функции и хранятся в `func.__defaults__`. Изменяемый объект будет разделяться между всеми вызовами.

```python
def bad(bar=[]):
    bar.append(1)
    return bar

bad()  # [1]
bad()  # [1, 1]  ← мутирует тот же список!
bad()  # [1, 1, 1]
```

**Решение:** использовать `None` как sentinel и создавать новый объект внутри:

```python
def good(bar=None):
    if bar is None:
        bar = []
    bar.append(1)
    return bar

good()  # [1]
good()  # [1]
```

То же справедливо для `dict`, `set` и любых других изменяемых объектов.

---

## Как передаются аргументы в функцию

Python использует механизм **"передача по ссылке на объект"** (pass by object reference / call by sharing).

- При вызове функции создаются новые **имена** (локальные переменные), которые связываются с теми же объектами, что и аргументы.
- Для **неизменяемых** типов (int, str, tuple): операции создают новый объект, локальное имя перепривязывается — оригинал не меняется.
- Для **изменяемых** типов (list, dict, set): мутирующие операции изменяют сам объект — изменения видны снаружи.

```python
def modify(lst, num):
    lst.append(4)   # мутирует объект → виден снаружи
    num += 1        # создаёт новый int → не виден снаружи

my_list = [1, 2, 3]
my_num = 10
modify(my_list, my_num)
print(my_list)  # [1, 2, 3, 4]
print(my_num)   # 10
```

---

## Функция как объект первого класса

Функции в Python — объекты первого класса: их можно присваивать, передавать как аргументы, возвращать из функций, хранить в коллекциях.

```python
def greet(name):
    return f'Hello, {name}'

# Присваивание
say_hi = greet

# Передача в функцию
def apply(func, value):
    return func(value)

apply(greet, 'World')  # 'Hello, World'

# Хранение в словаре
handlers = {'greet': greet}
```

---

## Вложенные функции

Функцию можно объявить внутри другой. Она видна только в теле внешней функции.

```python
def outer():
    def inner():
        return 42
    return inner()
```

---

## Лямбды

Анонимные функции, состоящие из одного выражения. Не резервируют имя в пространстве имён.

```python
square = lambda x: x ** 2
square(5)  # 25

# Часто используются с sorted, map, filter
pairs = [(1, 'b'), (2, 'a')]
sorted(pairs, key=lambda p: p[1])  # [(2, 'a'), (1, 'b')]
```

**Ограничения:**
- Только одно выражение (не statement)
- Нельзя использовать `pass`, `return`, `raise`, `assert` и другие операторы

```python
lambda: pass            # SyntaxError
lambda x: raise Ex(x)  # SyntaxError
```

---

## Замыкание (closure)

Вложенная функция, которая захватывает переменные из окружающей области видимости (enclosing scope). Каждый вызов внешней функции создаёт новый экземпляр замыкания с независимым состоянием.

```python
def make_counter(start=0):
    count = start

    def increment():
        nonlocal count
        count += 1
        return count

    return increment

c1 = make_counter()
c2 = make_counter(10)

c1()  # 1
c1()  # 2
c2()  # 11  ← независимое состояние
```

Замыкания — основа декораторов и фабрик функций. Для изменения переменной внешней функции нужен `nonlocal`.

---

## Декораторы

Синтаксический сахар для оборачивания функций. `@decorator` эквивалентно `func = decorator(func)`.

### Что может быть декоратором

Любой **вызываемый объект**: функция, лямбда, класс, экземпляр с `__call__`.

```python
# Функция-декоратор (стандартно)
def my_dec(func):
    def wrapper(*args, **kwargs): return func(*args, **kwargs)
    return wrapper

# Класс как декоратор
class retry:
    def __init__(self, times=3):
        self.times = times

    def __call__(self, func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(self.times):
                try:
                    return func(*args, **kwargs)
                except Exception:
                    pass
        return wrapper

@retry(times=3)
def flaky_request():
    ...
```

Применять декоратор можно к функциям, методам, классам.

### Что будет если декоратор не возвращает ничего

```python
def broken_decorator(func):
    def wrapper(*args, **kwargs):
        func(*args, **kwargs)
    # нет return wrapper!

@broken_decorator
def greet():
    print('hello')

greet   # None — функция заменена на None
greet() # TypeError: 'NoneType' object is not callable
```

Если декоратор возвращает `None` — декорируемый объект становится `None`.

### Разница @foobar vs @foobar()

```python
@foobar     # foobar — сам декоратор, принимает функцию
def view(): ...

@foobar()   # foobar() возвращает декоратор (фабрика декораторов)
def view(): ...
```

```python
import functools

def log(func):
    @functools.wraps(func)  # сохраняет __name__, __doc__ оригинала
    def wrapper(*args, **kwargs):
        print(f'Calling {func.__name__}')
        result = func(*args, **kwargs)
        print(f'Done')
        return result
    return wrapper

@log
def add(a, b):
    return a + b

add(1, 2)
# Calling add
# Done
```

Декоратор с параметрами — это фабрика, возвращающая декоратор:

```python
def repeat(n):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(n):
                func(*args, **kwargs)
        return wrapper
    return decorator

@repeat(3)
def hello():
    print('Hi')
```

### Вызов декоратора без @

`@decorator` — это синтаксический сахар. Всегда можно написать явно:

```python
def auth_only(view):
    def wrapper(request):
        if not request.user.is_authenticated:
            raise PermissionError
        return view(request)
    return wrapper

def dashboard(request):
    ...

dashboard = auth_only(dashboard)  # эквивалентно @auth_only
```

---

## functools

### functools.wraps

Декоратор для обёрток — копирует `__name__`, `__doc__`, `__module__`, `__qualname__` и `__wrapped__` из оборачиваемой функции. Без него декорированная функция в стектрейсах и `help()` отображается как `wrapper`.

```python
import functools

def my_decorator(func):
    @functools.wraps(func)  # обязательно для корректных стектрейсов
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def my_func():
    """Документация функции."""
    pass

my_func.__name__  # 'my_func' (без wraps было бы 'wrapper')
my_func.__doc__   # 'Документация функции.'
```

### functools.lru_cache

Декоратор-кэш для функций с детерминированным результатом. Хранит последние N вызовов (LRU — Least Recently Used). Аргументы должны быть хешируемыми.

```python
import functools

@functools.lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

fibonacci(100)  # мгновенно, без кэша — экспоненциальное время

# Статистика кэша
fibonacci.cache_info()   # CacheInfo(hits=98, misses=101, maxsize=128, currsize=101)
fibonacci.cache_clear()  # очистить кэш

# @cache — то же, но без ограничения размера (Python 3.9+)
@functools.cache
def fib(n): ...
```

### functools.partial

Частичное применение функции — создаёт новую функцию с предзаполненными аргументами.

```python
import functools

def power(base, exp):
    return base ** exp

square = functools.partial(power, exp=2)
cube   = functools.partial(power, exp=3)

square(5)  # 25
cube(3)    # 27

# Полезно с map/filter
double = functools.partial(int.__mul__, 2)
list(map(double, [1, 2, 3]))  # [2, 4, 6]
```
