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

---

## Race Condition и блокировки

**Race condition** — ситуация когда результат зависит от порядка выполнения потоков.

Защита: `threading.Lock` (мьютекс), `threading.RLock` (реентерабельный), `threading.Semaphore`, `queue.Queue` (потокобезопасная очередь).

```python
lock = threading.Lock()
with lock:
    shared_counter += 1  # атомарно
```

### Lock vs RLock

- `Lock` — можно захватить один раз, повторный захват из того же потока вызовет deadlock
- `RLock` (reentrant lock) — можно захватить несколько раз из одного потока, нужно столько же раз освободить

### Deadlock

Ситуация когда два или более потоков ждут друг друга и ни один не может продолжить. Классика: поток A захватил Lock 1 и ждёт Lock 2, поток B захватил Lock 2 и ждёт Lock 1.

Решения: захватывать блокировки в одном порядке, использовать `Lock.acquire(timeout=...)`, избегать вложенных блокировок.

---

## threading vs asyncio

- **asyncio** — тысячи I/O-операций (HTTP запросы, WebSocket), одного потока достаточно, async/await экосистема
- **threading** — десятки I/O-операций, интеграция с синхронным кодом, простые фоновые задачи
- **multiprocessing** — CPU-bound задачи (вычисления, обработка данных)

---

## Future

Объект из `concurrent.futures`, представляющий результат асинхронной операции. Можно проверить статус (`.done()`), получить результат (`.result()` — блокирующий), добавить callback (`.add_done_callback()`). Создаётся через `executor.submit()`.

```python
with ThreadPoolExecutor(max_workers=10) as pool:
    futures = [pool.submit(fetch, url) for url in urls]
    for f in as_completed(futures):
        result = f.result()
```

---

## Передача данных между процессами

- `Queue` — потокобезопасная очередь для передачи данных
- `Pipe` — двусторонний канал (быстрее Queue для двух процессов)
- `Manager` — shared state через прокси-объекты (медленно, удобно)
- `Value` / `Array` — shared memory для простых типов

---

## Daemon потоки и процессы

Фоновый поток/процесс, который автоматически завершается при выходе основного потока/процесса. Используется для фоновых задач (мониторинг, логирование).

```python
t = threading.Thread(target=worker)
t.daemon = True  # установить до start()
t.start()
```

---

## Потокобезопасность встроенных типов

В CPython `list.append()` безопасен благодаря GIL — один `append` это одна байткод-операция. Но полагаться на это не стоит: это деталь реализации CPython, не гарантия языка. Для надёжности используй `queue.Queue` или `Lock`.
