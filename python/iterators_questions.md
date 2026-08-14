# Python — Итераторы и генераторы: Вопросы

> Теория: [iterators.md](iterators.md)

---

**Q: Чем отличается Iterable от Iterator?**

- **Iterable** — объект с `__iter__()`, можно итерировать многократно (list, str, dict)
- **Iterator** — объект с `__iter__()` + `__next__()`, одноразовый, помнит позицию

```python
lst = [1, 2, 3]        # Iterable, но не Iterator
it = iter(lst)          # Iterator
isinstance(it, Iterable)  # True (Iterator ⊂ Iterable)
```

---

**Q: Что происходит в цикле `for x in obj`?**

1. Вызывается `iter(obj)` → получаем Iterator
2. В цикле вызывается `next(iterator)` → получаем элемент
3. Когда `next()` выбрасывает `StopIteration` → цикл завершается

---

**Q: Как создать свой итератор?**

Реализовать `__iter__()` и `__next__()`:

```python
class Counter:
    def __init__(self, max):
        self.max = max
        self.current = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.current >= self.max:
            raise StopIteration
        self.current += 1
        return self.current
```

---

**Q: Что такое генератор?**

Функция с `yield`, которая возвращает Iterator. При вызове не выполняется — создаёт объект генератора. Каждый `next()` выполняет код до следующего `yield`, возвращает значение и замораживает состояние.

```python
def count(n):
    for i in range(n):
        yield i

gen = count(3)  # объект генератора
next(gen)       # 0
next(gen)       # 1
```

---

**Q: Чем генератор лучше списка?**

- **Ленивость** — значения вычисляются по одному, не все сразу
- **Память** — O(1) вместо O(n) для больших последовательностей
- **Бесконечные последовательности** — возможны с генератором, невозможны со списком

---

**Q: Что такое generator expression?**

Генераторное выражение — ленивый аналог list comprehension:

```python
squares_list = [x**2 for x in range(1000000)]   # список в памяти
squares_gen  = (x**2 for x in range(1000000))    # генератор, O(1) память
```

---

**Q: Что делает `yield from`?**

Делегирует итерацию другому iterable/генератору. Эквивалент `for item in iterable: yield item`, но также прокидывает `send()`, `throw()`, `close()` и возвращаемое значение.

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)  # делегирует рекурсивно
        else:
            yield item
```

---

**Q: Что делает метод `send()` у генератора?**

Отправляет значение в генератор — оно становится результатом текущего `yield`:

```python
def accumulator():
    total = 0
    while True:
        value = yield total
        total += value

gen = accumulator()
next(gen)          # 0 (инициализация)
gen.send(10)       # 10
gen.send(20)       # 30
```

---

**Q: Что делают `throw()` и `close()` у генератора?**

- `gen.throw(ExcType)` — выбрасывает исключение в точке yield
- `gen.close()` — выбрасывает `GeneratorExit` в генератор, завершает его

---

**Q: Назовите полезные функции из `itertools`.**

- `chain(*iterables)` — конкатенация итераторов
- `islice(iterable, stop)` — срез итератора (ленивый)
- `groupby(iterable, key)` — группировка по ключу
- `product(A, B)` — декартово произведение
- `combinations(iterable, r)` — комбинации
- `permutations(iterable, r)` — перестановки
- `count(start)` — бесконечный счётчик
- `cycle(iterable)` — бесконечный цикл по итератору
- `zip_longest(*iterables)` — zip с заполнением коротких

---

**Q: Можно ли перезапустить генератор?**

Нет. Генератор — одноразовый итератор. После исчерпания нужно создать новый. Если нужно итерировать многократно — используй list или создавай генератор заново.

---

**Q: Чем `map()`/`filter()` отличаются от list comprehension?**

`map()` и `filter()` возвращают ленивые итераторы (как генераторы). List comprehension создаёт список сразу. Для ленивости с comprehension-синтаксисом — используй generator expression.

```python
map(str, range(10))          # ленивый итератор
[str(x) for x in range(10)] # список в памяти
(str(x) for x in range(10)) # ленивый генератор
```
