# JavaScript: Вопросы

> Теория: [javascript.md](javascript.md)

---

## Типы и сравнение

**Q: Какие типы данных есть в JavaScript?**

7 примитивов: `string`, `number`, `bigint`, `boolean`, `undefined`, `null`, `symbol`. Один ссылочный: `object` (включая Array, Function, Date, Map, Set и т.д.).

---

**Q: В чём разница `==` и `===`?**

- `==` — нестрогое сравнение, приводит типы перед сравнением (`0 == false` → `true`)
- `===` — строгое, без приведения типов (`0 === false` → `false`)

Всегда используй `===`. Исключение: `x == null` проверяет и `null`, и `undefined` одновременно.

---

**Q: Что такое NaN и как его проверить?**

`NaN` — результат некорректной числовой операции. Единственное значение, не равное самому себе (`NaN === NaN` → `false`). Проверка: `Number.isNaN(x)` (не `isNaN()` — тот приводит аргумент к числу).

---

**Q: Чем отличаются `null`, `undefined` и undeclared?**

- `null` — явное "нет значения", присваивается программистом
- `undefined` — переменная объявлена, но не инициализирована
- undeclared — переменная не объявлена, обращение вызовет `ReferenceError`

---

## Переменные и scope

**Q: `var` vs `let` vs `const` — в чём разница?**

| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisting | Да (→ `undefined`) | Да (TDZ) | Да (TDZ) |
| Повторное объявление | Да | Нет | Нет |
| Переприсваивание | Да | Да | Нет |

---

**Q: Что такое hoisting?**

Объявления `var` и `function declaration` поднимаются в начало области видимости. `var` поднимается как `undefined`, функция — целиком. `let`/`const` тоже поднимаются, но попадают в TDZ (Temporal Dead Zone) — обращение до инициализации вызовет `ReferenceError`.

---

**Q: Что такое TDZ (Temporal Dead Zone)?**

Зона от начала блока до строки объявления `let`/`const`. Переменная существует, но обращение к ней вызовет `ReferenceError`. Защищает от использования переменной до инициализации.

---

## Функции

**Q: Function Declaration vs Function Expression vs Arrow Function?**

- **Declaration** — поднимается (hoisting), можно вызвать до объявления
- **Expression** — не поднимается, присваивается переменной
- **Arrow** — нет своего `this`, `arguments`, `prototype`, нельзя использовать с `new`

---

**Q: Что такое замыкание (closure)?**

Функция, которая помнит переменные из своей лексической области видимости даже после завершения внешней функции. Позволяет создавать приватное состояние.

```js
function makeCounter() {
    let count = 0;
    return () => ++count;
}
const c = makeCounter();
c(); // 1
c(); // 2
```

---

**Q: Классический вопрос: что выведет `for (var i = 0; i < 3; i++) { setTimeout(() => console.log(i), 0); }`?**

Выведет `3 3 3`. `var` имеет function scope — одна переменная `i` на все итерации. К моменту выполнения callback'ов цикл завершился и `i === 3`. С `let` будет `0 1 2` — блочная область, каждая итерация получает свою копию.

---

## this

**Q: Как определяется `this` в JavaScript?**

По приоритету:
1. `new` → новый объект
2. `bind`/`call`/`apply` → явно указанный контекст
3. Метод объекта → сам объект
4. Обычный вызов → `undefined` (strict) или `globalThis` (non-strict)
5. Arrow function → `this` из лексического окружения (не своё)

---

**Q: Чем отличаются `call`, `apply` и `bind`?**

- `call(ctx, arg1, arg2)` — вызывает с указанным `this`, аргументы через запятую
- `apply(ctx, [args])` — то же, но аргументы массивом
- `bind(ctx)` — возвращает новую функцию с привязанным `this` (не вызывает)

---

**Q: Почему arrow function не подходит как метод объекта?**

Arrow function берёт `this` из лексического окружения (где была создана), а не из контекста вызова. Как метод объекта она не получит `this` объекта.

---

## Объекты и массивы

**Q: Shallow copy vs deep copy — в чём разница?**

- **Shallow** (`{...obj}`, `Object.assign`) — копирует только первый уровень, вложенные объекты — по ссылке
- **Deep** (`structuredClone()`, `JSON.parse(JSON.stringify())`) — копирует всю структуру рекурсивно

`structuredClone` — современный способ. `JSON.parse/stringify` не работает с функциями, `Date`, `undefined`, циклическими ссылками.

---

**Q: Какие методы массива мутируют оригинал, а какие нет?**

- **Мутируют:** `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`
- **Не мутируют:** `map`, `filter`, `reduce`, `slice`, `concat`, `flat`, `flatMap`

---

**Q: `map` vs `forEach` — в чём разница?**

`map` возвращает новый массив с результатами. `forEach` возвращает `undefined` — только для побочных эффектов. Если нужен результат — используй `map`.

---

## Event Loop и асинхронность

**Q: Как работает Event Loop?**

JS — однопоточный. Event Loop обеспечивает асинхронность:
1. Выполнить весь синхронный код (Call Stack)
2. Выполнить все микрозадачи (Promise.then, queueMicrotask) — до полного опустошения
3. Взять одну макрозадачу (setTimeout, setInterval, I/O)
4. Снова выполнить все микрозадачи
5. Повторить

---

**Q: Microtask vs Macrotask — в чём разница?**

- **Microtasks:** `Promise.then/catch/finally`, `queueMicrotask`, `MutationObserver`. Выполняются все до следующей макрозадачи.
- **Macrotasks:** `setTimeout`, `setInterval`, I/O callbacks, `setImmediate` (Node). Выполняется одна за итерацию.

---

**Q: Что выведет: `console.log(1); setTimeout(() => console.log(2), 0); Promise.resolve().then(() => console.log(3)); console.log(4);`?**

`1, 4, 3, 2`. Синхронный код (1, 4) → микрозадачи (Promise → 3) → макрозадачи (setTimeout → 2).

---

## Promises и async/await

**Q: Какие состояния у Promise?**

Три: `pending` (ожидание), `fulfilled` (успех), `rejected` (ошибка). После перехода из `pending` — состояние неизменяемо.

---

**Q: `Promise.all` vs `Promise.allSettled` vs `Promise.race` vs `Promise.any`?**

- `all` — ждёт всех, падает при первой ошибке
- `allSettled` — ждёт всех, возвращает статус каждого (никогда не падает)
- `race` — возвращает первый завершившийся (success или error)
- `any` — возвращает первый успешный, падает только если все упали

---

**Q: Почему `await` в цикле `for` — плохо? Как сделать параллельно?**

`await` в цикле выполняет запросы последовательно. Для параллельного выполнения — `Promise.all(ids.map(id => fetch(id)))`.

---

## Обработка ошибок

**Q: Как работает обработка ошибок в JavaScript?**

`try/catch/finally`:
- `try` — код, который может бросить ошибку
- `catch` — перехват ошибки
- `finally` — выполняется всегда (даже при `return` в try/catch)

```js
try {
    const data = JSON.parse(input);
} catch (err) {
    if (err instanceof SyntaxError) {
        console.error("Invalid JSON:", err.message);
    } else {
        throw err;  // перебросить неизвестную ошибку
    }
} finally {
    cleanup();  // выполнится в любом случае
}
```

---

**Q: Как обработать ошибку в промисе?**

Три способа:
- `.catch()` в цепочке: `fetch(url).then(handle).catch(err => ...)`
- `try/catch` с `async/await` — предпочтительный способ
- `Promise.allSettled()` — ждёт завершения всех промисов и возвращает массив `{status, value/reason}` для каждого, не прерываясь при ошибках

Необработанный rejected Promise → `unhandledRejection` → крэш в Node.js.

---

## Destructuring

**Q: Что такое деструктуризация?**

Извлечение значений из массивов и объектов в переменные:

```js
// Массив
const [a, b, ...rest] = [1, 2, 3, 4];  // a=1, b=2, rest=[3,4]

// Объект
const { name, age = 18, address: { city } } = user;

// В параметрах функции
function greet({ name, role = "user" }) { ... }

// Swap без временной переменной
[a, b] = [b, a];

// Переименование
const { name: userName } = user;  // userName = user.name
```

---

## Modules

**Q: `import`/`export` vs `require`/`module.exports` — в чём разница?**

- `import`/`export` (ESM) — статические, анализируются до выполнения, поддерживают tree-shaking, top-level `await`
- `require`/`module.exports` (CommonJS) — динамические, выполняются в runtime, синхронные

ESM — стандарт языка. CommonJS — legacy формат Node.js. В современном коде предпочтительнее ESM.

---

**Q: Что такое tree-shaking и почему он работает только с ESM?**

Удаление неиспользуемого кода при сборке (Webpack, Rollup). Работает с ESM потому что `import`/`export` — статические: бандлер на этапе компиляции знает какие экспорты используются. `require()` — динамический, бандлер не может определить что нужно.

---

## Прототипы

**Q: Как работает прототипное наследование?**

Каждый объект имеет скрытую ссылку `[[Prototype]]` на другой объект. При обращении к свойству JS ищет его сначала в самом объекте, затем по цепочке прототипов до `null`. `class` — синтаксический сахар над прототипами.

```
dog → Dog.prototype → Animal.prototype → Object.prototype → null
```

---

## Map / Set / WeakMap

**Q: Map vs Object — когда что?**

- `Map` — любые ключи (объекты, числа), сохраняет порядок, быстрее при частых add/delete, `.size`
- `Object` — только string/symbol ключи, удобнее для JSON, литеральный синтаксис

---

**Q: Зачем нужен WeakMap?**

Ключи — только объекты, слабые ссылки (не препятствуют GC). Нельзя итерировать, нет `.size`. Используется для: кэширования без утечек памяти, хранения метаданных объектов, приватных данных.

---

## Прочее

**Q: Что такое Symbol и зачем нужен?**

Уникальный примитив. Два символа с одинаковым описанием — всегда разные. Используется для: уникальных ключей объекта (не видны в `for...in`, `Object.keys`), well-known symbols (`Symbol.iterator`, `Symbol.toPrimitive`).

---

**Q: Что такое `?.` (optional chaining) и `??` (nullish coalescing)?**

- `?.` — безопасный доступ: возвращает `undefined` вместо ошибки если промежуточное значение `null`/`undefined`
- `??` — возвращает правый операнд только при `null`/`undefined` (в отличие от `||`, который срабатывает на любой falsy)

```js
const port = process.env.PORT ?? 3000;  // 0 не заменится (в отличие от ||)
```

---

**Q: Что такое генератор (function*)?**

Функция, которая может приостанавливать (`yield`) и возобновлять (`next()`) выполнение. Возвращает Iterator. Используется для: ленивых вычислений, бесконечных последовательностей, кастомных итераторов.

---

**Q: Что такое `yield*`?**

Делегирование итерации другому iterable/генератору. Эквивалент `for (const x of iterable) yield x`, но также прокидывает `next()`, `throw()`, `return()`.
