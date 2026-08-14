# Python — Основы: Вопросы

> Теория: [basics.md](basics.md)

---

**Q: Какие типы данных в Python являются изменяемыми (mutable), а какие — неизменяемыми (immutable)?**

- **Immutable:** `int`, `float`, `bool`, `str`, `tuple`, `frozenset`, `bytes`, `None`
- **Mutable:** `list`, `dict`, `set`, `bytearray`

Immutable объекты нельзя изменить после создания — любая "модификация" создаёт новый объект. Mutable объекты изменяются in-place.

---

**Q: Почему `tuple` нельзя использовать как ключ словаря, если он содержит list?**

Ключи словаря должны быть hashable. `tuple` hashable только если все его элементы hashable. `list` — не hashable (mutable), поэтому `tuple` с `list` внутри тоже не hashable.

```python
d = {}
d[(1, 2)] = 'ok'          # OK
d[(1, [2])] = 'fail'      # TypeError: unhashable type: 'list'
```

---

**Q: Чем отличается `list` от `tuple`?**

- `list` — mutable, можно менять элементы, добавлять, удалять. Используется для однородных коллекций.
- `tuple` — immutable, нельзя менять после создания. Hashable (если элементы hashable). Используется для разнородных фиксированных структур (координаты, записи).

---

**Q: Какая сложность операций у `list`?**

- `append` / `pop()` (с конца) — O(1) амортизированно
- `insert(0, x)` / `pop(0)` — O(n)
- `x in list` — O(n)
- `list[i]` — O(1)
- `sort` — O(n log n)

---

**Q: Какая сложность операций у `dict` и `set`?**

- Вставка, поиск, удаление — O(1) в среднем, O(n) в худшем (при коллизиях хешей)
- `dict` и `set` реализованы на хеш-таблицах
- `dict` сохраняет порядок вставки (с Python 3.7+)

---

**Q: Что такое hashable? Какие типы hashable?**

Hashable — объект, у которого есть метод `__hash__()` и который не меняется за время жизни. Все immutable встроенные типы hashable (`int`, `str`, `tuple` из hashable элементов, `frozenset`). Mutable типы (`list`, `dict`, `set`) — не hashable.

---

**Q: Чем `set` отличается от `frozenset`?**

- `set` — mutable, можно добавлять/удалять элементы, не hashable
- `frozenset` — immutable, hashable, можно использовать как ключ dict или элемент set

---

**Q: Что такое `defaultdict` и когда использовать?**

`defaultdict` из `collections` — словарь, который автоматически создаёт значение по умолчанию для отсутствующего ключа (через вызов фабричной функции).

```python
from collections import defaultdict
d = defaultdict(list)
d['key'].append(1)  # не нужен предварительный d['key'] = []
```

---

**Q: `OrderedDict` vs обычный `dict` — есть ли разница?**

С Python 3.7+ обычный `dict` сохраняет порядок вставки. `OrderedDict` по-прежнему полезен для: сравнения с учётом порядка (`OrderedDict == OrderedDict` учитывает порядок), метод `move_to_end()`.

---

**Q: Что такое `deque` и когда использовать вместо `list`?**

`collections.deque` — двусторонняя очередь. `append`/`pop` с обоих концов за O(1). Используй когда нужны операции с началом списка (`appendleft`, `popleft`) — у `list` это O(n).

---

**Q: Как работает `is` vs `==`?**

- `is` — проверяет идентичность (один и тот же объект в памяти, сравнивает `id`)
- `==` — проверяет равенство значений (вызывает `__eq__`)

```python
a = [1, 2]
b = [1, 2]
a == b  # True — одинаковые значения
a is b  # False — разные объекты
```

---

**Q: Что такое интернирование строк (string interning)?**

Python кэширует некоторые строки (короткие, идентификаторы) — одинаковые строки могут ссылаться на один объект. Поэтому `'hello' is 'hello'` может быть `True`, но это деталь реализации — никогда не используй `is` для сравнения строк.

---

**Q: Что такое `*args` и `**kwargs`?**

- `*args` — tuple из позиционных аргументов
- `**kwargs` — dict из именованных аргументов

```python
def f(*args, **kwargs):
    print(args)    # (1, 2)
    print(kwargs)  # {'a': 3}

f(1, 2, a=3)
```

---

**Q: Что такое list comprehension? Когда лучше использовать обычный цикл?**

List comprehension — компактный способ создания списка: `[expr for x in iterable if cond]`. Быстрее обычного цикла (оптимизация CPython). Используй обычный цикл если: логика сложная, нужны побочные эффекты, вложенность > 2.

---

**Q: Что такое walrus operator `:=`?**

Оператор присваивания внутри выражения (Python 3.8+):

```python
# Вместо
data = input()
while data != 'quit':
    process(data)
    data = input()

# С walrus
while (data := input()) != 'quit':
    process(data)
```

---

**Q: Чем `bytes` отличается от `str`?**

- `str` — последовательность Unicode-символов
- `bytes` — последовательность байтов (0-255)

Преобразование: `str.encode('utf-8') → bytes`, `bytes.decode('utf-8') → str`.
