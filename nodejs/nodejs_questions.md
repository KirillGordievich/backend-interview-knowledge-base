# Node.js Runtime: Вопросы

> Теория: [nodejs.md](nodejs.md)

---

## Архитектура

**Q: Что такое Node.js и из чего он состоит?**

Однопоточный JavaScript runtime на движке V8. Ключевые компоненты:
- **V8** — компиляция и выполнение JS + сборка мусора
- **libuv** — кроссплатформенный event loop + thread pool
- **Node.js API** — встроенные модули (fs, http, crypto, ...)

---

**Q: Что такое libuv и зачем нужен thread pool?**

libuv — C-библиотека, реализующая event loop и асинхронный I/O. Thread pool (по умолчанию 4 потока) выполняет блокирующие операции не блокируя event loop:
- Файловая система (`fs.*`)
- DNS lookups (`dns.lookup`)
- Crypto операции
- Сжатие (zlib)

Сетевые операции (HTTP, TCP) идут через OS kernel (epoll/kqueue), без thread pool.

---

## Event Loop

**Q: Какие фазы у Event Loop в Node.js?**

6 фаз:
1. **timers** — `setTimeout`, `setInterval`
2. **pending callbacks** — I/O ошибки из предыдущей итерации
3. **idle, prepare** — внутреннее использование
4. **poll** — ожидание I/O событий, выполнение I/O callbacks
5. **check** — `setImmediate`
6. **close callbacks** — `socket.on('close')`

Между каждой фазой выполняются все `process.nextTick` и Promise microtasks.

---

**Q: `process.nextTick` vs `setImmediate` vs `setTimeout(fn, 0)` — порядок?**

1. `process.nextTick` — после текущей операции, до всего остального (самый высокий приоритет)
2. Promise microtasks — после nextTick
3. `setTimeout(fn, 0)` — фаза timers
4. `setImmediate` — фаза check (после I/O)

**Важно:** рекурсивный `process.nextTick` может заморозить event loop — предпочитай `setImmediate` для разбивки тяжёлых задач.

---

**Q: Чем Event Loop в Node.js отличается от браузерного?**

В браузере — упрощённая модель (microtask queue + macrotask queue). В Node.js — 6 фаз с разным приоритетом. Есть `process.nextTick` и `setImmediate` (нет в браузере). Нет Web APIs (setTimeout реализован через libuv).

---

## Модули

**Q: CommonJS vs ESM — в чём разница?**

| | CommonJS | ESM |
|---|---|---|
| Синтаксис | `require`/`module.exports` | `import`/`export` |
| Загрузка | Синхронная | Асинхронная |
| Tree-shaking | Нет | Да |
| Top-level await | Нет | Да |
| `__dirname` | Есть | Нет (использовать `import.meta.url`) |

CommonJS — legacy, большинство npm пакетов. ESM — современный стандарт.

---

**Q: Как включить ESM в Node.js проекте?**

Два способа:
1. Расширение `.mjs` для файлов
2. `"type": "module"` в `package.json` (все `.js` файлы станут ESM)

---

## EventEmitter

**Q: Что такое EventEmitter и когда использовать?**

Встроенный паттерн Observer. Объект эмитит именованные события, подписчики реагируют. Используется для: развязки компонентов, уведомлений о событиях, streaming.

```js
emitter.on('event', handler)     // подписка
emitter.once('event', handler)   // одноразовая подписка
emitter.emit('event', data)      // синхронно вызывает все обработчики
```

**Важно:** необработанное событие `'error'` = `throw` → крэш процесса.

---

## Streams

**Q: Что такое Streams и зачем нужны?**

Обработка данных по частям (chunks) без загрузки всего в память. 4 типа:
- `Readable` — источник (fs, HTTP request, stdin)
- `Writable` — приёмник (fs, HTTP response, stdout)
- `Transform` — преобразование (gzip, шифрование)
- `Duplex` — двунаправленный (TCP socket)

---

**Q: `pipe` vs `pipeline` — в чём разница?**

- `stream.pipe(dest)` — простой, но не обрабатывает ошибки корректно (утечки)
- `pipeline(src, transform, dest)` — корректно обрабатывает ошибки и cleanup. Всегда используй `pipeline` из `stream/promises`.

---

## Buffer

**Q: Что такое Buffer?**

Представляет сырые бинарные данные фиксированного размера. Хранится вне V8 heap. Используется для: работы с файлами, сетевыми протоколами, бинарными данными.

```js
Buffer.from('hello', 'utf8')
buf.toString('base64')
```

---

**Q: `Buffer.alloc` vs `Buffer.allocUnsafe` — в чём разница?**

- `alloc(size)` — заполняет нулями, безопасно
- `allocUnsafe(size)` — быстрее, но может содержать старые данные из памяти. Используй только если сразу перезапишешь содержимое.

---

## Масштабирование

**Q: Cluster vs Worker Threads — когда что?**

| | Cluster | Worker Threads |
|---|---|---|
| Изоляция | Отдельные процессы | Потоки одного процесса |
| Память | Не разделяется | Общая через SharedArrayBuffer |
| Коммуникация | IPC (медленно) | `postMessage` + SharedArrayBuffer |
| Когда | HTTP сервер (масштабирование I/O) | CPU-heavy задачи (парсинг, crypto) |

---

**Q: Как масштабировать Node.js приложение?**

1. **Cluster** — несколько процессов на одном сервере, OS распределяет запросы
2. **PM2** — менеджер процессов (cluster mode + мониторинг + рестарт)
3. **Горизонтальное** — load balancer (nginx) + несколько инстансов/контейнеров
4. **Worker Threads** — для CPU-bound задач в отдельных потоках

---

## Process

**Q: Как обработать необработанные исключения в Node.js?**

```js
process.on('uncaughtException', (err) => {
    console.error('Uncaught:', err);
    process.exit(1);   // всегда завершать!
});

process.on('unhandledRejection', (reason) => {
    console.error('Unhandled rejection:', reason);
    process.exit(1);
});
```

**Важно:** после `uncaughtException` состояние приложения ненадёжно — нужно завершать процесс.

---

**Q: Что такое graceful shutdown?**

Корректное завершение: перестать принимать новые запросы, дождаться завершения текущих, закрыть соединения (DB, Redis), и только потом выйти.

```js
process.on('SIGTERM', async () => {
    await server.close();
    await db.disconnect();
    process.exit(0);
});
```

---

**Q: Node.js — однопоточный. Как он обрабатывает тысячи запросов?**

Event loop + неблокирующий I/O. Пока один запрос ждёт ответа от БД (I/O), Node обрабатывает другие запросы. Потоки не блокируются — вместо этого callback'и/промисы ставятся в очередь. Для CPU-bound задач — Worker Threads или вынести в отдельный сервис.
