# Python — Многопоточность и многопроцессорность

## GIL (Global Interpreter Lock)

### Что такое GIL

GIL — блокировка, которая позволяет только **одному потоку** выполнять байткод Python в любой момент времени.

**Принцип работы:**
1. Поток захватывает GIL и выполняется
2. Через ~100 инструкций или при операции I/O — отпускает GIL
3. Другой поток захватывает GIL и продолжает
4. При I/O поток явно освобождает GIL и блокируется в ожидании

```
Поток 1: ====GIL====|  ждёт  |====GIL====|  ждёт  |
Поток 2: | ждёт  |====GIL====| ждёт  |====GIL====|
```

### Зачем нужен GIL

- **Безопасность reference counting:** подсчёт ссылок (`ob_refcnt`) не атомарен. Без GIL два потока могут одновременно изменить счётчик и либо не удалить объект, либо удалить живой.
- **Простота C-расширений:** библиотеки на C не обязаны быть thread-safe. Это причина богатой экосистемы C-биндингов в Python.
- **Защита изменяемых структур:** словари, списки не thread-safe. GIL обеспечивает атомарность операций на уровне байткода.

### Последствия GIL

- **CPU-bound задачи в потоках не ускоряются** — потоки выполняются последовательно даже на многоядерном CPU
- **I/O-bound задачи выигрывают** — при ожидании I/O GIL отпускается, другой поток работает
- Переключение контекста между потоками — дорогая операция (сброс регистров, кэша)

### Замена GIL

В Python 3.13 появился экспериментальный режим `--without-gil` (Free-threaded Python, PEP 703), но он ещё не является стандартным.

---

## Потоки (Threading)

Нативные POSIX-треды, управляемые ОС. Используют общее адресное пространство.

```python
import threading

results = []
lock = threading.Lock()

def worker(n):
    result = n * 2
    with lock:        # защита общего ресурса
        results.append(result)

threads = [threading.Thread(target=worker, args=(i,)) for i in range(10)]
for t in threads:
    t.start()
for t in threads:
    t.join()          # ждать завершения всех
```

**Когда использовать:**
- I/O-bound задачи в синхронном коде (запросы к API, файлы, БД)
- Когда нельзя использовать asyncio (блокирующие библиотеки)

**Когда не использовать:**
- CPU-bound задачи — из-за GIL ускорения нет, есть накладные расходы

### Примитивы threading

```python
threading.Lock()       # взаимное исключение
threading.RLock()      # реентерантный Lock (тот же поток может захватить повторно)
threading.Event()      # сигнал между потоками
threading.Semaphore(n) # ограничение параллелизма
threading.Barrier(n)   # ждать пока n потоков достигнут точки
```

---

## Процессы (Multiprocessing)

Каждый процесс — независимый интерпретатор Python с собственным GIL и памятью. Позволяет использовать все ядра CPU.

```python
from multiprocessing import Pool

def heavy_compute(n):
    return sum(i * i for i in range(n))

with Pool(processes=4) as pool:
    results = pool.map(heavy_compute, [10**6, 10**6, 10**6, 10**6])
```

**Когда использовать:**
- CPU-bound задачи: вычисления, обработка данных, ML

**Минусы:**
- Запуск процесса — дорого (копирование памяти, запуск интерпретатора)
- Обмен данными только через IPC (очереди, pipes, shared memory)

### fork()

`os.fork()` создаёт копию текущего процесса. Дочерний процесс наследует память родителя (copy-on-write), но работает независимо.

```python
import os

pid = os.fork()
if pid == 0:
    print('Дочерний процесс')
else:
    print(f'Родительский процесс, PID дочернего: {pid}')
```

Копия процесса начинает чуть больше весить при записи в память (copy-on-write становится реальным копированием).

---

## Что выбрать

| Задача | Решение |
|---|---|
| CPU-bound (вычисления) | `multiprocessing` |
| I/O-bound, синхронный код | `threading` |
| I/O-bound, асинхронный код | `asyncio` |
| Распределённые задачи | Celery, RQ |

```
Нужна параллельность?
├── CPU-bound → multiprocessing (обходит GIL через отдельные процессы)
└── I/O-bound → threading или asyncio
    ├── Синхронные библиотеки → threading
    └── Асинхронные библиотеки → asyncio (меньше overhead)
```

### Пример: 100 вычислительных задач

Использовать потоки — плохо. GIL заставит их работать последовательно, а переключение контекста добавит накладные расходы. Правильно — `multiprocessing.Pool` или `concurrent.futures.ProcessPoolExecutor`.

```python
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor

# CPU-bound
with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(cpu_task, data))

# I/O-bound
with ThreadPoolExecutor(max_workers=20) as executor:
    results = list(executor.map(io_task, urls))
```
