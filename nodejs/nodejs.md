# Node.js Runtime

## Архитектура

Node.js — однопоточный JavaScript runtime на движке V8. Асинхронность реализована через **libuv** — кроссплатформенную библиотеку event loop и thread pool.

```
┌─────────────────────────────────────────────┐
│              Node.js Process                │
│                                             │
│  ┌──────────────┐    ┌────────────────────┐ │
│  │  V8 Engine   │    │       libuv        │ │
│  │ (JS execution│    │  ┌──────────────┐  │ │
│  │  + GC)       │    │  │  Event Loop  │  │ │
│  └──────────────┘    │  └──────────────┘  │ │
│                      │  ┌──────────────┐  │ │
│  ┌──────────────┐    │  │ Thread Pool  │  │ │
│  │  Node.js API │    │  │ (4 потока)   │  │ │
│  │ (fs, crypto, │    │  └──────────────┘  │ │
│  │  http, ...)  │    └────────────────────┘ │
│  └──────────────┘                           │
└─────────────────────────────────────────────┘
```

**Thread Pool (libuv)** выполняет тяжёлые операции не блокируя event loop:
- Файловая система (`fs.*`)
- DNS lookups (`dns.lookup`)
- Crypto операции (`bcrypt`, `pbkdf2`)
- Сжатие (`zlib`)

Сетевые операции (HTTP, TCP) — через OS kernel (epoll/kqueue), без thread pool.

---

## Event Loop фазы

В Node.js event loop имеет **6 фаз** (в отличие от браузера):

```
   ┌───────────────────────────┐
┌─►│          timers           │  setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │     pending callbacks     │  I/O ошибки из предыдущей итерации
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │       idle, prepare       │  внутреннее использование
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │           poll            │  ждёт I/O событий, выполняет I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │           check           │  setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
└──│      close callbacks      │  socket.on('close', ...)
   └───────────────────────────┘
```

**Между каждой фазой** выполняются все `process.nextTick` и Promise microtasks.

---

## setImmediate vs process.nextTick vs setTimeout(fn, 0)

```js
console.log("1 — sync");

process.nextTick(() => console.log("2 — nextTick"));

Promise.resolve().then(() => console.log("3 — microtask"));

setImmediate(() => console.log("4 — setImmediate"));

setTimeout(() => console.log("5 — setTimeout 0"), 0);

console.log("6 — sync");

// Порядок: 1, 6, 2, 3, 5, 4
// (nextTick → microtasks → timers phase → check phase)
```

| | Когда выполняется | Приоритет |
|---|---|---|
| `process.nextTick` | После текущей операции, до всего остального | Самый высокий |
| `Promise.then` | Microtask queue | После nextTick |
| `setTimeout(fn, 0)` | Фаза timers | После microtasks |
| `setImmediate` | Фаза check (после I/O) | После timers |

**Практическое правило:** `process.nextTick` — рекурсивный вызов может заморозить event loop. Предпочитай `setImmediate` для разбивки тяжёлых задач.

**Event Loop: Node.js vs браузер** — в браузере упрощённая модель (microtask queue + macrotask queue). В Node.js — 6 фаз с разным приоритетом. Есть `process.nextTick` и `setImmediate` (нет в браузере). Нет Web APIs (setTimeout реализован через libuv).

---

## CommonJS vs ESM

### CommonJS (`.js`, `.cjs`)

```js
// module.exports / require — синхронные, кэшируются
const path = require('path');
const { readFile } = require('fs');

module.exports = { greet };
module.exports.greet = greet;   // то же самое

// Динамический импорт
const config = require(`./config/${env}`);
```

### ESM — ES Modules (`.mjs` или `"type": "module"` в package.json)

```js
// import/export — статические, анализируются до выполнения
import path from 'path';
import { readFile } from 'fs/promises';
import * as utils from './utils.js';

export const greet = (name) => `Hello, ${name}`;
export default class User { ... }

// Динамический импорт (async)
const config = await import(`./config/${env}.js`);
```

| | CommonJS | ESM |
|---|---|---|
| Синтаксис | `require`/`module.exports` | `import`/`export` |
| Загрузка | Синхронная | Асинхронная |
| Tree-shaking | Нет | Да |
| Top-level await | Нет | Да |
| `__dirname` | Да | Нет (использовать `import.meta.url`) |
| Когда | Node.js legacy, большинство пакетов | Современные проекты |

```js
// __dirname в ESM
import { fileURLToPath } from 'url';
import { dirname } from 'path';
const __filename = fileURLToPath(import.meta.url);
const __dirname  = dirname(__filename);
```

**Как включить ESM:** расширение `.mjs` для файлов, или `"type": "module"` в `package.json` (все `.js` станут ESM).

---

## EventEmitter

Паттерн Observer встроенный в Node.js.

```js
import { EventEmitter } from 'events';

class OrderService extends EventEmitter {
    async placeOrder(order) {
        await db.save(order);
        this.emit('order:created', order);         // синхронно вызывает все обработчики
        this.emit('order:notification', order.userId, order.id);
    }
}

const service = new OrderService();

service.on('order:created', (order) => {
    console.log(`Order #${order.id} created`);
});

service.once('order:created', (order) => {   // сработает только один раз
    analytics.track('first_order', order);
});

service.on('error', (err) => {               // всегда обрабатывай 'error'!
    console.error('OrderService error:', err);
});

service.removeAllListeners('order:created'); // удалить все обработчики
service.listenerCount('order:created');      // 0
```

**Важно:** необработанное событие `'error'` = `throw` → крэш процесса.

---

## Streams

Позволяют обрабатывать данные **по частям** (chunks) — без загрузки всего в память.

```
Readable → Transform → Writable
```

```js
import { createReadStream, createWriteStream } from 'fs';
import { createGzip } from 'zlib';
import { pipeline } from 'stream/promises';

// Сжать файл через pipe
await pipeline(
    createReadStream('large-file.txt'),
    createGzip(),
    createWriteStream('large-file.txt.gz')
);

// Читать по строкам
import { createInterface } from 'readline';

const rl = createInterface({
    input: createReadStream('data.csv'),
    crlfDelay: Infinity
});

for await (const line of rl) {
    processLine(line);   // обрабатываем строку за строкой
}

// Transform stream
import { Transform } from 'stream';

const upperCase = new Transform({
    transform(chunk, encoding, callback) {
        this.push(chunk.toString().toUpperCase());
        callback();
    }
});

process.stdin.pipe(upperCase).pipe(process.stdout);
```

**`pipe` vs `pipeline`:** `stream.pipe(dest)` — простой, но не обрабатывает ошибки корректно (утечки). `pipeline(src, transform, dest)` из `stream/promises` — корректно обрабатывает ошибки и cleanup. Всегда используй `pipeline`.

**Типы streams:**
- `Readable` — источник данных (fs, HTTP request, stdin)
- `Writable` — приёмник (fs, HTTP response, stdout)
- `Transform` — преобразование (gzip, cipher, parse)
- `Duplex` — двунаправленный (TCP socket)

---

## Buffer

Представляет сырые бинарные данные (фиксированного размера, в памяти вне V8 heap).

```js
// Создание
const buf1 = Buffer.from('hello', 'utf8');
const buf2 = Buffer.from([0x68, 0x65, 0x6c, 0x6c, 0x6f]);
const buf3 = Buffer.alloc(10);           // 10 нулевых байт
const buf4 = Buffer.allocUnsafe(10);     // быстрее, но может содержать мусор

// Чтение
buf1.toString('utf8');   // "hello"
buf1.toString('hex');    // "68656c6c6f"
buf1.toString('base64'); // "aGVsbG8="

// Операции
Buffer.concat([buf1, buf2]);
buf1.slice(1, 3);   // "el"
buf1.length;        // 5

// Buffer vs TypedArray
// Buffer — Node.js специфичный, удобен для I/O
// TypedArray — Web стандарт (Uint8Array), работает в браузере тоже
```

---

## Cluster и Worker Threads

### Cluster — несколько процессов на одном сервере

```js
import cluster from 'cluster';
import { cpus } from 'os';
import http from 'http';

if (cluster.isPrimary) {
    const numCPUs = cpus().length;
    console.log(`Primary ${process.pid} is running`);

    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();   // создать дочерний процесс
    }

    cluster.on('exit', (worker, code) => {
        console.log(`Worker ${worker.process.pid} died, restarting...`);
        cluster.fork();   // автоматический рестарт
    });
} else {
    // Каждый воркер слушает один и тот же порт — OS распределяет запросы
    http.createServer((req, res) => {
        res.end(`Worker ${process.pid}`);
    }).listen(3000);
}
```

**Cluster vs PM2:** PM2 делает то же самое + мониторинг, рестарт, логи.

### Worker Threads — многопоточность для CPU-задач

```js
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

if (isMainThread) {
    // Главный поток
    const worker = new Worker('./worker.js', {
        workerData: { numbers: [1, 2, 3, 4, 5] }
    });

    worker.on('message', (result) => console.log('Result:', result));
    worker.on('error', (err) => console.error(err));
    worker.on('exit', (code) => console.log(`Exit: ${code}`));
} else {
    // Поток-воркер
    const { numbers } = workerData;
    const sum = numbers.reduce((a, b) => a + b, 0);
    parentPort.postMessage(sum);
}
```

| | Cluster | Worker Threads |
|---|---|---|
| Изоляция | Отдельные процессы | Потоки одного процесса |
| Память | Не разделяется | Общая через SharedArrayBuffer |
| Коммуникация | IPC (медленно) | `postMessage` + SharedArrayBuffer |
| Когда | HTTP сервер (I/O) | CPU-heavy (парсинг, crypto, ML) |

**Масштабирование Node.js:**
1. **Cluster** — несколько процессов на одном сервере, OS распределяет запросы
2. **PM2** — менеджер процессов (cluster mode + мониторинг + рестарт)
3. **Горизонтальное** — load balancer (nginx) + несколько инстансов/контейнеров
4. **Worker Threads** — для CPU-bound задач в отдельных потоках

**"Node.js однопоточный — как обрабатывает тысячи запросов?"** — Event loop + неблокирующий I/O. Пока один запрос ждёт ответа от БД (I/O), Node обрабатывает другие. Потоки не блокируются — callback'и/промисы ставятся в очередь. Для CPU-bound задач — Worker Threads или отдельный сервис.

---

## process объект

```js
// Переменные среды
process.env.NODE_ENV;     // "production"
process.env.PORT;         // "3000"

// Информация о процессе
process.pid;              // ID процесса
process.ppid;             // ID родительского процесса
process.platform;         // "linux", "darwin", "win32"
process.version;          // "v20.11.0"
process.memoryUsage();    // { rss, heapTotal, heapUsed, external }
process.cpuUsage();

// Аргументы командной строки
process.argv;             // ["node", "app.js", "--port", "3000"]

// Завершение
process.exit(0);          // нормальное завершение
process.exit(1);          // ошибка

// Обработка необработанных исключений
process.on('uncaughtException', (err) => {
    console.error('Uncaught:', err);
    process.exit(1);   // всегда завершать процесс!
});

process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled rejection at:', promise, 'reason:', reason);
    process.exit(1);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
    console.log('SIGTERM received, shutting down...');
    await server.close();
    await db.disconnect();
    process.exit(0);
});
```
