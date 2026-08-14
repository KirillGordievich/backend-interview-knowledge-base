# Python — Concurrency: Вопросы

> Теория: [concurrency.md](concurrency.md)

---

**Q: Что такое GIL и зачем он нужен?**

Global Interpreter Lock — мьютекс в CPython, который позволяет только одному потоку исполнять Python-байткод одновременно. Нужен для защиты внутренних структур CPython (подсчёт ссылок) от race conditions. Из-за GIL многопоточность в Python не даёт ускорения на CPU-bound задачах.

---

**Q: GIL блокирует всё? Когда потоки реально работают параллельно?**

GIL отпускается при I/O операциях (сетевые запросы, чтение файлов, sleep) и при вызове C-расширений (numpy, PIL). Поэтому threading полезен для I/O-bound задач — потоки реально работают параллельно на I/O, хотя Python-код выполняется по очереди.

---

**Q: Чем отличается threading от multiprocessing?**

| | `threading` | `multiprocessing` |
|---|---|---|
| Память | Общая (один процесс) | Раздельная (отдельные процессы) |
| GIL | Да, ограничивает CPU | Нет, каждый процесс — свой GIL |
| Лучше для | I/O-bound (запросы, файлы) | CPU-bound (вычисления) |
| Overhead | Низкий | Высокий (fork/spawn, IPC) |
| Общение | Shared state, Lock | Queue, Pipe, Manager |

---

**Q: Что такое race condition и как от него защититься?**

Race condition — ситуация когда результат зависит от порядка выполнения потоков. Защита: `threading.Lock` (мьютекс), `threading.RLock` (реентерабельный), `threading.Semaphore`, `queue.Queue` (потокобезопасная очередь).

```python
lock = threading.Lock()
with lock:
    shared_counter += 1  # атомарно
```

---

**Q: Чем `Lock` отличается от `RLock`?**

- `Lock` — можно захватить один раз, повторный захват из того же потока вызовет deadlock
- `RLock` (reentrant lock) — можно захватить несколько раз из одного потока, нужно столько же раз освободить

---

**Q: Что такое deadlock?**

Ситуация когда два или более потоков ждут друг друга и ни один не может продолжить. Классика: поток A захватил Lock 1 и ждёт Lock 2, поток B захватил Lock 2 и ждёт Lock 1.

Решения: захватывать блокировки в одном порядке, использовать `Lock.acquire(timeout=...)`, избегать вложенных блокировок.

---

**Q: threading vs asyncio — когда что выбирать?**

- **asyncio** — тысячи I/O-операций (HTTP запросы, WebSocket), одного потока достаточно, async/await экосистема
- **threading** — десятки I/O-операций, интеграция с синхронным кодом, простые фоновые задачи
- **multiprocessing** — CPU-bound задачи (вычисления, обработка данных)

---

**Q: Что такое `ThreadPoolExecutor` и `ProcessPoolExecutor`?**

Пулы потоков/процессов из `concurrent.futures`. Удобный интерфейс: `submit()` возвращает `Future`, `map()` для batch-обработки, `as_completed()` для обработки по мере готовности.

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=10) as pool:
    futures = [pool.submit(fetch, url) for url in urls]
    for f in as_completed(futures):
        result = f.result()
```

---

**Q: Что такое `Future` в Python?**

Объект, представляющий результат асинхронной операции. Можно проверить статус (`.done()`), получить результат (`.result()` — блокирующий), добавить callback (`.add_done_callback()`). Создаётся `executor.submit()`.

---

**Q: Как передавать данные между процессами в multiprocessing?**

- `Queue` — потокобезопасная очередь для передачи данных
- `Pipe` — двусторонний канал (быстрее Queue для двух процессов)
- `Manager` — shared state через прокси-объекты (медленно, удобно)
- `Value` / `Array` — shared memory для простых типов

---

**Q: Что такое daemon thread/process?**

Фоновый поток/процесс, который автоматически завершается при выходе основного потока/процесса. Используется для фоновых задач (мониторинг, логирование). Устанавливается: `thread.daemon = True` перед `start()`.

---

**Q: Безопасен ли `list.append()` в многопоточном Python?**

В CPython — да, благодаря GIL. Один `append` — одна байткод-операция. Но полагаться на это не стоит: это деталь реализации CPython, не гарантия языка. Для надёжности используй `queue.Queue` или `Lock`.

---

**Q: Что изменится с отменой GIL (PEP 703)?**

Python 3.13+ вводит экспериментальный free-threaded mode (no-GIL). Потоки смогут исполнять Python-код параллельно. Для защиты данных придётся использовать явные блокировки. Пока экспериментальный — нужно включать явно при сборке.
