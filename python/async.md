# Python — Asyncio

## Основы

### Когда использовать asyncio

Asyncio подходит для задач, где программа большую часть времени **ожидает** внешних операций (сеть, БД, диск), а не выполняет вычисления:

- Много HTTP-запросов к внешним сервисам
- WebSockets, long polling
- Медленные запросы к базам данных

Для CPU-bound задач asyncio не даёт выигрыша — используйте `multiprocessing`.

---

### Цикл событий (Event Loop)

Ядро asyncio. Хранит список всех текущих задач и управляет их выполнением.

Упрощённая модель работы:
1. Цикл проходит по задачам, вызывая `next(task)` для каждой
2. Задача выполняется до ближайшего `await`
3. При `await` задача уступает управление циклу (через `yield` внутри)
4. Цикл переходит к следующей задаче
5. Когда ожидаемая операция завершается — задача ставится обратно в очередь

Цикл событий asyncio написан на C (использует `libuv`/`epoll`/`kqueue` в зависимости от ОС).

```python
import asyncio

async def main():
    await asyncio.sleep(1)

asyncio.run(main())  # создаёт цикл, запускает, закрывает
```

---

## Корутины и задачи

### Корутина (Coroutine)

Функция, объявленная через `async def`. При вызове возвращает объект-корутину — не выполняется сразу. Выполнить можно только через `await` или добавив в цикл событий.

```python
async def fetch(url):
    # await приостанавливает корутину, отдаёт управление циклу
    response = await http_client.get(url)
    return response

# Вызов без await — просто создаёт объект корутины, не запускает:
coro = fetch('https://example.com')  # ничего не произошло

# Правильно:
result = await fetch('https://example.com')
```

### Task (Задача)

`asyncio.Task` — обёртка над корутиной, которая **планирует** её выполнение в цикле событий независимо. Подкласс `Future`.

```python
async def main():
    # create_task планирует выполнение немедленно
    task = asyncio.create_task(fetch('https://example.com'))

    # можно делать другую работу пока task выполняется
    await asyncio.sleep(0)

    result = await task  # дождаться результата
```

**Разница корутина vs Task:**
- Корутина — объект, который надо добавить в цикл чтобы он выполнился
- Task — уже запланированная корутина, выполняется независимо, можно ждать результат

### Запуск нескольких задач параллельно

```python
async def main():
    # gather — запустить все и дождаться всех результатов
    results = await asyncio.gather(
        fetch('https://api1.com'),
        fetch('https://api2.com'),
        fetch('https://api3.com'),
    )

    # TaskGroup (Python 3.11+) — более безопасная альтернатива
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(fetch('https://api1.com'))
        task2 = tg.create_task(fetch('https://api2.com'))
    # здесь оба task завершены, при ошибке одного — отменяются остальные
```

### await и __await__

`await expr` работает с любым объектом, у которого есть метод `__await__()`. Он должен возвращать итератор. Встроенные awaitable: корутины, `Task`, `Future`, объекты с `__await__`.

---

## Примитивы синхронизации

### asyncio.Lock

Взаимное исключение для asyncio-задач. Пример — защита кэша: не допустить N параллельных запросов к БД за одним и тем же значением.

```python
cache = {}
lock = asyncio.Lock()

async def get_data(key):
    async with lock:
        if key not in cache:
            cache[key] = await fetch_from_db(key)
    return cache[key]
```

### asyncio.Event

Сигнал между задачами. Управляет внутренним флагом (`True`/`False`).

```python
event = asyncio.Event()

async def waiter():
    await event.wait()   # блокируется пока флаг False
    print('event triggered!')

async def trigger():
    await asyncio.sleep(1)
    event.set()           # устанавливает флаг True, будит всех ожидающих

async def main():
    await asyncio.gather(waiter(), waiter(), trigger())
```

### asyncio.Semaphore

Ограничивает количество одновременно выполняемых операций. Внутренний счётчик уменьшается при `acquire()` и увеличивается при `release()`. При достижении 0 — блокирует.

```python
# Не более 5 параллельных запросов
sem = asyncio.Semaphore(5)

async def limited_fetch(url):
    async with sem:
        return await fetch(url)

async def main():
    urls = ['https://example.com'] * 100
    results = await asyncio.gather(*[limited_fetch(u) for u in urls])
```

---

## uvloop

Замена стандартному циклу событий asyncio, написанная на Cython поверх `libuv`. Даёт прирост производительности в 2-4x для сетевых задач.

```python
import uvloop

asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
# или
uvloop.run(main())
```

---

## Uvicorn

ASGI-сервер для запуска асинхронных Python-приложений (FastAPI, Starlette и др.). Реализует спецификацию ASGI (Asynchronous Server Gateway Interface).

```bash
uvicorn app:app --reload          # разработка
uvicorn app:app --workers 4       # продакшн (несколько процессов)
uvicorn app:app --loop uvloop     # с uvloop для скорости
```

Для продакшна обычно запускается за nginx или через gunicorn с uvicorn workers:

```bash
gunicorn app:app -w 4 -k uvicorn.workers.UvicornWorker
```
