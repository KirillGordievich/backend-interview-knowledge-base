# Python — Internals

## Замер времени

### perf_counter vs time()

`time.time()` возвращает текущее системное время (wall clock). На него влияют внешние события: смена часового пояса, синхронизация NTP, переход на летнее время.

`time.perf_counter()` — монотонный счётчик высокого разрешения. Не привязан к системным часам, не подвержен их изменениям. Используется для замера **длительности** участков кода.

```python
import time

start = time.perf_counter()
# ... код ...
elapsed = time.perf_counter() - start
print(f'{elapsed:.6f} сек')
```

`perf_counter()` не имеет абсолютного значения — важна только разность двух вызовов.

Для микробенчмарков используйте `timeit`:

```python
import timeit

timeit.timeit('"-".join(str(n) for n in range(100))', number=10000)
```

---

## Управление памятью

### Как Python выделяет память

- **pymalloc** — собственный аллокатор для объектов до 512 байт. Быстрее `malloc`, минимизирует фрагментацию.
- **malloc** — для больших объектов (системный аллокатор).

Вся память выделяется из **heap** (кучи). Переменные в Python — это имена, связанные с объектами в куче. Стека для хранения значений нет.

### Reference Counting

Основной механизм GC. Каждый объект хранит счётчик ссылок (`ob_refcnt`). Когда счётчик падает до 0 — объект немедленно удаляется.

```python
import sys

x = [1, 2, 3]
sys.getrefcount(x)  # 2 (x + аргумент getrefcount)

y = x
sys.getrefcount(x)  # 3

del y
sys.getrefcount(x)  # 2
```

**Преимущества:** детерминированное удаление, нет stop-the-world пауз для большинства объектов.
**Проблема:** не обнаруживает **циклические ссылки**.

### Cyclic Garbage Collector

Дополняет reference counting для обнаружения циклических ссылок.

**Поколения (Generational GC):**
- Объекты делятся на 3 поколения (0, 1, 2)
- Новые объекты — в поколении 0
- Выжившие после сборки переходят в следующее поколение
- Поколение 0 собирается часто (~каждые 700 аллокаций), поколения 1 и 2 — реже

```python
import gc

gc.collect()         # принудительная сборка
gc.disable()         # отключить (если нет циклических ссылок)
gc.get_count()       # (объектов в gen0, gen1, gen2)
gc.get_threshold()   # пороговые значения для запуска сборки
```

### Утечки памяти

Основные причины:
1. **Циклические ссылки с `__del__`** — GC не всегда может собрать объекты с финализаторами
2. **Глобальные переменные/кэши** — объекты держатся глобальным scope
3. **Закрытые ресурсы** — незакрытые файлы, соединения
4. **Расширения на C** — ошибки в reference counting

```python
# Диагностика утечек
import tracemalloc

tracemalloc.start()
# ... код ...
snapshot = tracemalloc.take_snapshot()
stats = snapshot.statistics('lineno')
for stat in stats[:5]:
    print(stat)
```

### Интернирование объектов

Python кэширует некоторые объекты для экономии памяти:
- Целые числа от -5 до 256
- Строки-идентификаторы (без пробелов и спецсимволов) при компиляции

```python
a = 256; b = 256; a is b  # True (интернированы)
a = 257; b = 257; a is b  # False (новые объекты)

s1 = 'hello'; s2 = 'hello'; s1 is s2  # True (интернированы)
s1 = 'hello world'; s2 = 'hello world'
s1 is s2  # неопределённо (зависит от реализации)
```

---

## PyObject — структура любого объекта

Все объекты в CPython — это структуры языка C. Базовая структура:

```c
// Include/object.h (упрощённо)
typedef struct _object {
    Py_ssize_t ob_refcnt;   // счётчик ссылок
    PyTypeObject *ob_type;  // указатель на тип (класс)
} PyObject;

// Объекты переменной длины (list, tuple, str, bytes):
typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size;     // количество элементов
} PyVarObject;
```

```python
import sys

# Каждый объект занимает минимум 28 байт (64-bit): refcnt + type + GC header
sys.getsizeof(object())   # 16 (без GC overhead)
sys.getsizeof(42)          # 28 (PyLongObject)
sys.getsizeof([])          # 56 (PyListObject: header + pointer array)
sys.getsizeof({})          # 64 (PyDictObject: header + hash table)
sys.getsizeof("")          # 49 (PyUnicodeObject: header + kind + hash)
```

---

## Реализация dict в CPython

### Compact Dict (Python 3.6+)

До 3.6 dict был обычной хеш-таблицей: массив из `(hash, key, value)` с 1/3–2/3 пустых слотов → огромный расход памяти.

С 3.6 — **compact dict**: два массива вместо одного.

```
Indices (sparse):  [None, 1, None, 0, None, None, 2, None]
                          ↓              ↓                    ↓
Entries (dense):   [(hash_a, key_a, val_a),    ← index 0
                    (hash_b, key_b, val_b),    ← index 1
                    (hash_c, key_c, val_c)]    ← index 2
```

- **Indices** — разреженный массив (1-8 байт на ячейку), хранит индексы в entries
- **Entries** — плотный массив `(hash, key, value)`, без дырок
- Экономия ~25% памяти по сравнению с Python 3.5
- **Порядок вставки сохраняется** (гарантия с Python 3.7)

### Вставка ключа

```python
# Упрощённый алгоритм:
def dict_insert(table, key, value):
    h = hash(key)
    idx = h % len(table.indices)     # начальный индекс (slot)

    while table.indices[idx] is not EMPTY:
        entry_idx = table.indices[idx]
        entry = table.entries[entry_idx]

        if entry.hash == h and entry.key == key:
            entry.value = value      # обновить существующий
            return

        # Коллизия — probing
        idx = next_probe(idx, h)

    # Свободный слот найден
    table.indices[idx] = len(table.entries)
    table.entries.append((h, key, value))
```

### Коллизии и Probing

**Коллизия** — два ключа попадают в один слот: `hash(a) % size == hash(b) % size`.

CPython использует **open addressing** с **perturbed probing** (не линейный и не квадратичный):

```c
// Objects/dictobject.c (упрощённо)
perturb = hash;
idx = hash & mask;           // mask = table_size - 1

while (slot_is_occupied) {
    // Проверить: hash совпадает И ключ равен?
    if (entry->hash == hash && entry->key == key) {
        return entry;  // нашли
    }
    // Следующий probe: комбинация перемешивания всех бит хеша
    perturb >>= 5;
    idx = (5 * idx + perturb + 1) & mask;
}
```

**Почему не линейное пробирование:** линейное создаёт кластеры (clustering) — цепочки занятых слотов, замедляющие поиск. Perturbed probing использует все биты хеша для "прыжков" по таблице → равномерное распределение.

### Resize (перестроение)

```python
# Порог: если заполнено > 2/3 слотов → resize
# Рост: примерно x2 (точнее: до следующей степени 2 × 2/3 ≥ used)
# При resize ВСЕ элементы перехешируются (O(n))

# Размеры таблицы: 8, 16, 32, 64, 128, 256, ... (степени двойки)
# Минимальный размер: 8 слотов
```

### `__hash__` + `__eq__` взаимодействие

```python
# Контракт:
# 1. a == b → hash(a) == hash(b)           (ОБЯЗАТЕЛЬНО)
# 2. hash(a) == hash(b) → a может != b     (коллизия — нормально)
# 3. Нарушение п.1 ломает dict/set:

class Broken:
    def __init__(self, x):
        self.x = x
    def __eq__(self, other):
        return self.x == other.x
    def __hash__(self):
        return id(self)   # ОШИБКА: a == b, но hash(a) != hash(b)

a, b = Broken(1), Broken(1)
d = {a: "hello"}
print(d[b])   # KeyError! Хеши разные → Python ищет в другом слоте
```

```python
# Правильно:
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __eq__(self, other):
        return isinstance(other, Point) and (self.x, self.y) == (other.x, other.y)
    def __hash__(self):
        return hash((self.x, self.y))  # ← зависит от тех же полей, что __eq__
```

---

## Реализация set в CPython

`set` — хеш-таблица **без значений** (хранит только ключи). Тот же механизм: open addressing + perturbed probing.

```c
// Структура записи set (entry):
typedef struct {
    Py_hash_t hash;
    PyObject *key;      // только ключ, без value
} setentry;
```

Операции `in`, `add`, `discard` — O(1) амортизировано (как dict).

`frozenset` — то же, но immutable + хешируемый (можно использовать как ключ dict или элемент set).

---

## Реализация list в CPython

### Dynamic Array (динамический массив)

```c
// Include/cpython/listobject.h (упрощённо)
typedef struct {
    PyObject_VAR_HEAD
    PyObject **ob_item;    // указатель на массив указателей на объекты
    Py_ssize_t allocated;  // выделенных слотов (>= ob_size)
} PyListObject;
```

```
ob_item → [ptr, ptr, ptr, ptr, NULL, NULL, NULL, NULL]
           │    │    │    │
           0    1    2    3   ← ob_size = 4, allocated = 8
```

### Over-allocation (предвыделение)

При `append` Python **не выделяет ровно +1** — он выделяет с запасом, чтобы следующие append были O(1).

```c
// Objects/listobject.c — формула роста:
// new_allocated = (newsize >> 3) + (newsize < 9 ? 3 : 6)
// → рост примерно: 0, 4, 8, 16, 24, 32, 40, 52, 64, 76, ...
```

```python
import sys

lst = []
prev = sys.getsizeof(lst)
for i in range(20):
    lst.append(i)
    size = sys.getsizeof(lst)
    if size != prev:
        print(f"len={len(lst):2d}  allocated_bytes={size}  +{size - prev}")
        prev = size

# len= 1  allocated_bytes=88   +32   (было 56 → выделил на 4 элемента)
# len= 5  allocated_bytes=120  +32   (перевыделил на 8)
# len= 9  allocated_bytes=184  +64   (перевыделил на 16)
# len=17  allocated_bytes=248  +64   (перевыделил на ~24)
```

### Почему insert(0, x) — O(n)

```
До:    [A, B, C, D, _, _]

insert(0, X):
  → сдвинуть [A, B, C, D] вправо на 1 → O(n)
  → [X, A, B, C, D, _]

append(X):
  → записать в конец → O(1)
  → [A, B, C, D, X, _]
```

Для O(1) вставки/удаления с обоих концов — `collections.deque` (двусвязный список блоков).

---

## Устройство генераторов в CPython

### Frame Object (фрейм вызова)

Каждый вызов функции создаёт **frame object** — структуру, хранящую состояние выполнения.

```c
// Include/cpython/frameobject.h (упрощённо)
typedef struct _frame {
    PyObject_HEAD
    struct _frame *f_back;   // предыдущий фрейм (call stack)
    PyCodeObject *f_code;    // объект кода (байткод)
    PyObject *f_locals;      // локальные переменные
    PyObject *f_globals;     // глобальные переменные
    int f_lasti;             // индекс последней выполненной инструкции
    // ... и другие поля
} PyFrameObject;
```

**Обычная функция:** frame создаётся при вызове → выполняется → уничтожается при return.
**Генератор:** frame **замораживается** при `yield` и **размораживается** при `next()`.

### Атрибуты генератора

```python
def gen_func(x):
    y = x + 1
    yield y
    y += 10
    yield y

g = gen_func(5)

# gi_frame — текущий фрейм выполнения (None после завершения)
print(g.gi_frame)            # <frame at 0x...>
print(g.gi_frame.f_locals)   # {} (ещё не начал выполнение)
print(g.gi_frame.f_lasti)    # -1 (ни одна инструкция не выполнена)

# gi_code — объект кода (байткод генератора)
print(g.gi_code)             # <code object gen_func at 0x...>
print(g.gi_code.co_varnames) # ('x', 'y')

# gi_running — True если генератор сейчас выполняется
print(g.gi_running)          # False

# gi_yieldfrom — подгенератор (при yield from)
print(g.gi_yieldfrom)        # None

next(g)  # → 6
print(g.gi_frame.f_locals)   # {'x': 5, 'y': 6}
print(g.gi_frame.f_lasti)    # индекс после yield

next(g)  # → 16
print(g.gi_frame.f_locals)   # {'x': 5, 'y': 16}

try:
    next(g)
except StopIteration:
    pass

print(g.gi_frame)  # None — фрейм освобождён после завершения
```

### Механизм suspend/resume

```
1. gen_func(5) → создаёт GeneratorObject + Frame, НО не выполняет код
   gi_frame.f_lasti = -1

2. next(g) → resume:
   - Восстановить gi_frame как текущий фрейм интерпретатора
   - Продолжить выполнение с f_lasti
   - Дойти до yield y → suspend:
     - Сохранить все локальные переменные в f_locals
     - Запомнить f_lasti (позицию в байткоде)
     - Вернуть yielded значение
   - gi_frame сохраняется (НЕ удаляется)

3. next(g) → resume снова с точки yield → дойти до следующего yield

4. next(g) → resume → дойти до конца функции → raise StopIteration
   - gi_frame = None (освобождён)
```

### Байткод генератора

```python
import dis

def gen():
    x = 1
    yield x
    x += 1
    yield x

dis.dis(gen)
#   RESUME          0
#   LOAD_CONST      1 (1)
#   STORE_FAST      0 (x)
#   LOAD_FAST       0 (x)
#   YIELD_VALUE     1        ← точка suspend
#   RESUME          1        ← точка resume после next()
#   POP_TOP
#   LOAD_FAST       0 (x)
#   LOAD_CONST      1 (1)
#   BINARY_OP       13 (+=)
#   STORE_FAST      0 (x)
#   LOAD_FAST       0 (x)
#   YIELD_VALUE     1        ← вторая точка suspend
#   RESUME          1
#   POP_TOP
#   RETURN_CONST    0 (None) ← неявный return None → StopIteration
```

`YIELD_VALUE` — сохраняет состояние фрейма и возвращает значение.
`RESUME` — восстанавливает состояние после вызова `next()`.

### Почему генераторы эффективнее

```python
# Список: создаёт все элементы в памяти сразу
squares_list = [x**2 for x in range(1_000_000)]  # ~8MB

# Генератор: хранит только один фрейм (~200 байт) + текущее значение
squares_gen = (x**2 for x in range(1_000_000))

import sys
sys.getsizeof(squares_list)  # 8448728
sys.getsizeof(squares_gen)   # 200  ← только GenObject + Frame
```

---

## type — метакласс

`type` — встроенный метакласс. Все классы в Python — экземпляры `type`.

```python
type(int)    # <class 'type'>
type(list)   # <class 'type'>
type(42)     # <class 'int'>

# type(name, bases, dict) — динамическое создание класса
MyClass = type('MyClass', (object,), {'x': 42, 'hello': lambda self: 'Hi'})
```

```python
class Foo: pass
type(Foo)              # <class 'type'>
type(type)             # <class 'type'> — type является своим же экземпляром
isinstance(Foo, type)  # True
```

Подробнее о метаклассах — см. [oop.md](oop.md).

---

## `__del__` — финализатор

Вызывается когда объект уничтожается. **Не рекомендуется к использованию:**
- Неопределённое время вызова (зависит от GC)
- Может задержать сборку циклических ссылок (GC не может собрать объекты с `__del__`, образующие цикл, до Python 3.4)
- Исключения внутри `__del__` игнорируются

```python
class Resource:
    def __del__(self):
        print("удаляется")  # может быть вызван в любой момент

# Правильная альтернатива — контекстный менеджер:
class Resource:
    def __enter__(self):
        return self
    def __exit__(self, *args):
        self.close()  # детерминированное освобождение ресурса
```

Используй `with` / контекстные менеджеры вместо `__del__`.

---

## `__sizeof__` и `sys.getsizeof`

`sys.getsizeof(obj)` — размер объекта в байтах **без учёта объектов, на которые он ссылается**.

Вызывает `obj.__sizeof__()` и добавляет overhead GC-заголовка.

```python
import sys

sys.getsizeof([1, 2, 3])      # 88 — размер list-объекта, не элементов
sys.getsizeof("hello")        # 54 — PyUnicodeObject + символы
sys.getsizeof({"a": 1})       # 232 — PyDictObject

# Для подсчёта "глубокого" размера нужна рекурсия:
def deep_size(obj, seen=None):
    if seen is None:
        seen = set()
    obj_id = id(obj)
    if obj_id in seen:
        return 0
    seen.add(obj_id)
    size = sys.getsizeof(obj)
    if isinstance(obj, dict):
        size += sum(deep_size(k, seen) + deep_size(v, seen) for k, v in obj.items())
    elif hasattr(obj, '__iter__') and not isinstance(obj, (str, bytes)):
        size += sum(deep_size(i, seen) for i in obj)
    return size
```

---

## set vs list: почему `in` быстрее для set

`set` использует хеш-таблицу: проверка `x in s` — O(1) в среднем (вычислить хеш, проверить слот).

`list` выполняет линейный перебор: `x in lst` — O(n).

```python
import timeit

data_list = list(range(100_000))
data_set = set(range(100_000))

timeit.timeit(lambda: 99_999 in data_list, number=1000)  # ~ms
timeit.timeit(lambda: 99_999 in data_set,  number=1000)  # ~мкс
```

Для частых проверок принадлежности всегда используй `set`. `frozenset` — то же, но immutable.
