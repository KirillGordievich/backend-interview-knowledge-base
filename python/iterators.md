# Python — Итераторы и генераторы

## Протоколы: Iterable → Iterator → Generator

Три уровня абстракции, каждый следующий расширяет предыдущий:

```
Iterable ─────────────── __iter__()
   ↑
Iterator ─────────────── __iter__() + __next__()
   ↑
Generator ────────────── __iter__() + __next__() + send() + throw() + close()
```

```python
from collections.abc import Iterable, Iterator, Generator
import typing

# Проверка:
isinstance([1, 2], Iterable)     # True  (есть __iter__)
isinstance([1, 2], Iterator)     # False (нет __next__)
isinstance(iter([1, 2]), Iterator)  # True

def gen(): yield 1
isinstance(gen(), Generator)     # True
isinstance(gen(), Iterator)      # True  (Generator IS-A Iterator)

# typing-аннотация генератора:
# Generator[YieldType, SendType, ReturnType]
def my_gen() -> Generator[int, str, bool]:
    value = yield 42     # yield int, receive str
    return True           # return bool
```

---

## Контейнер

Объект, инкапсулирующий значения других типов. Поддерживает `__contains__()`.
Примеры: `list`, `tuple`, `dict`, `set`, `str`.

---

## Iterable (итерабельный объект)

Объект, из которого можно получить итератор. Должен реализовывать `__iter__()` или `__getitem__()`.

```python
class MyIterable:
    def __iter__(self):
        return iter([1, 2, 3])

for x in MyIterable():  # работает
    print(x)
```

Функция `iter(obj)`:
1. Сначала вызывает `obj.__iter__()`
2. Если не реализован — пробует создать итератор через `obj.__getitem__()` (с индексами от 0)
3. Если ни того ни другого — `TypeError`

```python
class LegacySeq:
    def __getitem__(self, i):
        if i >= 3:
            raise IndexError
        return i * 10

for x in LegacySeq():  # 0, 10, 20
    print(x)
```

**Iterable ≠ Iterator:** iterable создаёт новый итератор при каждом вызове `__iter__()`. Это позволяет проходить по объекту несколько раз.

```python
class Repeatable:
    def __init__(self, data):
        self.data = data

    def __iter__(self):
        return iter(self.data)  # каждый раз новый итератор

r = Repeatable([1, 2, 3])
list(r)  # [1, 2, 3]
list(r)  # [1, 2, 3] ← работает повторно
```

---

## Iterator (итератор)

Объект, реализующий протокол итератора:
- `__next__()` — возвращает следующий элемент или поднимает `StopIteration`
- `__iter__()` — возвращает `self` (итератор сам является итерабельным)

```python
class Countdown:
    def __init__(self, n):
        self.n = n

    def __iter__(self):
        return self

    def __next__(self):
        if self.n <= 0:
            raise StopIteration
        self.n -= 1
        return self.n + 1

list(Countdown(3))  # [3, 2, 1]
```

**Важно:** итератор — одноразовый. После исчерпания он навсегда остаётся пустым. Итерабельный объект можно итерировать несколько раз — каждый раз создаётся новый итератор.

```python
lst = [1, 2, 3]
it = iter(lst)

next(it)  # 1
next(it)  # 2

# lst можно итерировать снова, it — нет
```

**Паттерн: разделение Iterable и Iterator:**
```python
class SensorData:
    """Iterable — можно проходить несколько раз."""
    def __init__(self, readings):
        self.readings = readings

    def __iter__(self):
        return SensorIterator(self.readings)  # новый итератор каждый раз

class SensorIterator:
    """Iterator — одноразовый, хранит позицию."""
    def __init__(self, readings):
        self.readings = readings
        self.index = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.index >= len(self.readings):
            raise StopIteration
        value = self.readings[self.index]
        self.index += 1
        return value
```

---

## Генератор

Объект типа `types.GeneratorType`, реализующий протокол итератора. Создаётся двумя способами:

1. **Генераторная функция** — функция с `yield`
2. **Генераторное выражение** — `(expr for x in iterable)`

Генератор хранит не все элементы, а только **внутреннее состояние** для вычисления следующего. Идеален для больших или бесконечных последовательностей.

---

## Генераторная функция

Функция с `yield` возвращает объект-генератор при вызове. Тело не выполняется сразу — только при первом `next()`.

```python
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

fib = fibonacci()
next(fib)  # 0
next(fib)  # 1
next(fib)  # 1
next(fib)  # 2
```

`yield` замораживает выполнение функции и возвращает значение. При следующем `next()` выполнение продолжается с того же места.

---

## return в генераторе и StopIteration.value

Когда генераторная функция завершается — автоматически поднимается `StopIteration`. Если есть `return value` — значение сохраняется в `StopIteration.value`.

```python
def gen_with_return():
    yield 1
    yield 2
    return "finished"   # НЕ возвращает значение через yield!

g = gen_with_return()
print(next(g))  # 1
print(next(g))  # 2

try:
    next(g)
except StopIteration as e:
    print(e.value)  # "finished" ← значение return
```

**Если `return` не написан — Python подставляет неявный `return None`:**
```python
def gen_no_return():
    yield 1

g = gen_no_return()
next(g)  # 1

try:
    next(g)
except StopIteration as e:
    print(e.value)  # None ← неявный return None
```

**Где используется StopIteration.value?** В `yield from`:
```python
def sub():
    yield 1
    yield 2
    return "sub_result"

def main():
    result = yield from sub()  # result = "sub_result"
    print(f"Got: {result}")
    yield 99

g = main()
print(next(g))  # 1
print(next(g))  # 2
print(next(g))  # Got: sub_result → 99
```

`yield from` автоматически ловит `StopIteration` от подгенератора и извлекает `.value`. Без `yield from` пришлось бы вручную ловить `StopIteration`.

**Важно:** `return` с значением в генераторе — это НЕ yield. Значение нельзя получить через `next()`, только через `StopIteration.value` или `yield from`.

---

## Генераторное выражение vs списковое включение

```python
# Списковое включение — создаёт список в памяти сразу
squares_list = [x**2 for x in range(1000000)]  # ~8 MB

# Генераторное выражение — ленивое, почти не занимает память
squares_gen = (x**2 for x in range(1000000))   # ~120 bytes

# Синтаксис одинаковый, только [] vs ()
sum(x**2 for x in range(100))  # скобки можно опустить внутри вызова
```

---

## yield как выражение (двусторонний канал)

`yield` может не только отдавать значения, но и принимать через `send()`:

```python
def accumulator():
    total = 0
    while True:
        value = yield total  # получаем значение и отдаём накопленное
        if value is None:
            break
        total += value

gen = accumulator()
next(gen)        # 0 — запуск (первый next обязателен, send(None))
gen.send(10)     # 10
gen.send(20)     # 30
gen.send(5)      # 35
```

---

## yield from (делегирование подгенератору)

`yield from iterable` — делегирует итерацию другому генератору/итерабельному объекту.

```python
def chain(*iterables):
    for it in iterables:
        yield from it

list(chain([1, 2], [3, 4], [5]))  # [1, 2, 3, 4, 5]
```

Преимущество перед `for x in sub: yield x`:
- прозрачно передаёт `send()` и `throw()` в подгенератор
- возвращаемое значение (`return`) подгенератора становится результатом `yield from`

```python
def sub():
    yield 1
    yield 2
    return 'done'

def main():
    result = yield from sub()  # result = 'done'
    print(result)
```

---

## Методы генераторов

| Метод | Описание |
|---|---|
| `__next__()` | Следующее значение или `StopIteration` |
| `send(value)` | Отправить значение как результат текущего `yield`, получить следующий `yield` |
| `throw(exc)` | Бросить исключение в точку приостановки |
| `close()` | Бросить `GeneratorExit` для завершения генератора |

```python
def gen():
    try:
        yield 1
        yield 2
    except GeneratorExit:
        print('Cleaning up...')

g = gen()
next(g)   # 1
g.close() # Cleaning up...
```

---

## Получить список из генератора

```python
gen = (x**2 for x in range(5))
lst = list(gen)  # [0, 1, 4, 9, 16]

# После этого gen исчерпан
list(gen)  # []
```

---

## Можно ли извлечь элемент генератора по индексу

Нет — генератор не поддерживает `__getitem__`. Нужно конвертировать в список или использовать `itertools.islice`:

```python
import itertools

gen = (x**2 for x in range(100))
third = next(itertools.islice(gen, 2, None))  # элемент с индексом 2 = 4
```

---

## Что возвращает итерация по словарю

Ключи. В Python 3.7+ порядок гарантирован — порядок вставки.

```python
d = {'a': 1, 'b': 2, 'c': 3}

for key in d:              # итерация по ключам
    print(key)

for key, val in d.items(): # итерация по парам
    print(key, val)
```

---

## itertools — полезные инструменты

```python
import itertools

# Бесконечные итераторы
itertools.count(10)          # 10, 11, 12, ...
itertools.cycle([1, 2, 3])   # 1, 2, 3, 1, 2, 3, ...
itertools.repeat(0, 5)       # 0, 0, 0, 0, 0

# Комбинаторика
itertools.combinations('ABC', 2)   # AB, AC, BC
itertools.permutations('AB', 2)    # AB, BA
itertools.product('AB', repeat=2)  # AA, AB, BA, BB

# Прочее
itertools.chain([1,2], [3,4])       # 1, 2, 3, 4
itertools.islice(range(100), 5)     # 0, 1, 2, 3, 4
itertools.groupby(sorted_iterable, key)
```
