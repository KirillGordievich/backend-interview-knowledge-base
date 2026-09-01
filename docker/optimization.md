# Docker — Оптимизация образов

## Зачем вообще оптимизировать Docker-образ?

Чем меньше образ — тем быстрее он пушится в registry и пуллится на сервер, а значит быстрее деплой. При автоскейлинге каждый новый под начинается с pull образа — и разница между 1 GB и 150 MB становится ощутимой. Ещё меньший образ содержит меньше пакетов, а значит и меньше потенциальных уязвимостей (CVE).

---

## Какой базовый образ выбрать?

| Образ | Размер | Когда использовать |
|---|---|---|
| `python:3.12` | ~900 MB | Нужны системные пакеты, C-расширения |
| `python:3.12-slim` | ~150 MB | Большинство Python-приложений |
| `python:3.12-alpine` | ~50 MB | Минимальный размер, но musl вместо glibc |
| `node:22` | ~1 GB | Нужны нативные модули с C-биндингами |
| `node:22-alpine` | ~130 MB | Большинство Node.js приложений |
| `node:22-slim` | ~200 MB | Когда alpine не подходит (нужна glibc) |
| `distroless` | ~20 MB | Только runtime, без shell и пакетного менеджера |

**Про Alpine и Python:** Alpine основан на musl libc, а не glibc. Многие Python-пакеты (numpy, pandas, psycopg2) не имеют готовых бинарных wheel для musl — они компилируются из C-исходников. Это делает сборку медленнее, а итоговый образ может оказаться даже больше, чем `slim`. Поэтому для Python обычно лучше `slim`, а для Node.js — `alpine` работает отлично.

---

## Зачем пиннить версии базовых образов?

Если написать `FROM alpine:latest` или даже `FROM node:22-alpine`, то при обновлении базового образа в Docker Hub твоя сборка может сломаться или получить неожиданные изменения. `latest` — это антипаттерн для production.

```dockerfile
# ПЛОХО — latest может поменяться в любой момент
FROM alpine:latest

# ХОРОШО — фиксируем minor-версию и версию Alpine
FROM node:22.15-alpine3.20

# ИДЕАЛЬНО для production — фиксируем по SHA digest
FROM node:22-alpine@sha256:abcdef123456...
```

Digest гарантирует, что образ побайтово идентичен тому, на котором ты тестировал. Minor-версия — разумный компромисс для большинства проектов.

---

## Что такое multi-stage build и зачем он нужен?

Это Dockerfile с несколькими `FROM`. Идея: на первой стадии (builder) ставим компиляторы, dev-зависимости и собираем проект. На второй стадии (runner) берём чистый базовый образ и копируем туда только результат сборки. Всё остальное (исходники, dev-пакеты, кеши) остаётся в builder-стадии и не попадает в финальный образ.

```dockerfile
# === Builder: ставим всё, собираем ===
FROM node:22-alpine AS builder
WORKDIR /app

COPY package.json pnpm-lock.yaml .npmrc ./
RUN npm install -g pnpm@9
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
RUN pnpm prune --prod   # удаляем devDependencies

# === Runner: только результат сборки ===
FROM node:22-alpine
WORKDIR /app

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

USER node
CMD ["node", "dist/main.js"]
```

В builder-стадии были TypeScript, ESLint, тесты, компиляторы — но ничего из этого нет в финальном образе. Только `dist/` и prod-зависимости.

**Для Go ещё эффектнее** — финальный образ может быть ~10 MB:

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM scratch
COPY --from=builder /app/server /server
CMD ["/server"]
```

Go собирается в один бинарник, поэтому можно использовать `scratch` — образ вообще без ОС.

**Для Nuxt/Next (standalone output)** — можно вообще без `node_modules` в runner:

```dockerfile
FROM node:22-slim AS runner
WORKDIR /app
COPY --from=builder /app/.output ./.output
CMD ["node", ".output/server/index.mjs"]
```

Nuxt nitro генерирует standalone-бандл, в который уже включены все зависимости. Runner-образ получается минимальным.

---

## Почему порядок COPY важен для скорости сборки?

Docker кеширует каждый слой (каждую инструкцию). Если слой не менялся — Docker берёт его из кеша. Но если слой изменился — все последующие слои тоже пересобираются. Это называется инвалидация кеша.

Установка зависимостей — самая долгая операция. Если скопировать весь код до `install`, то любое изменение в любом файле инвалидирует слой с зависимостями.

```dockerfile
# ПЛОХО — поменял одну строку в коде → зависимости ставятся заново
COPY . .
RUN pnpm install

# ХОРОШО — зависимости кешируются отдельно от кода
COPY package.json pnpm-lock.yaml .npmrc ./
RUN pnpm install --frozen-lockfile
COPY . .
```

Сначала копируем только `package.json` и lock-файл, ставим зависимости. Потом копируем код. Если код изменился, но зависимости нет — слой с `pnpm install` берётся из кеша.

---

## Зачем нужен `--frozen-lockfile`?

Флаг `--frozen-lockfile` (для pnpm) или `npm ci` (для npm) гарантирует, что пакетный менеджер поставит **ровно те версии**, которые записаны в lock-файле. Без этого флага, если в `package.json` написано `"lodash": "^4.17.0"`, менеджер может поставить 4.18.0, хотя в lock-файле была 4.17.21. Это ломает детерминистичность сборки.

```dockerfile
# pnpm
RUN pnpm install --frozen-lockfile

# npm — npm ci делает то же самое: ставит строго по lock-файлу
RUN npm ci
```

---

## Зачем нужен `.dockerignore`?

Когда Docker собирает образ, он отправляет содержимое директории (build context) в Docker daemon. Если в проекте лежит `node_modules` на 500 MB и папка `.git` на 200 MB — всё это передаётся при каждой сборке, даже если в `COPY` ты их не используешь. `.dockerignore` исключает файлы из build context.

```
node_modules
.git
.env
dist
.nuxt
.output
coverage
*.log
Dockerfile
docker-compose.yml
.dockerignore
```

Без `.dockerignore` команда `COPY . .` скопирует и `node_modules`, и `.env` с секретами, и `.git` с историей. Даже если потом удалить — файлы останутся в промежуточном слое образа.

---

## Зачем объединять RUN-инструкции?

Каждый `RUN` создаёт отдельный слой в образе. Если ставить пакеты отдельными командами — получаешь лишние слои, каждый из которых хранит промежуточное состояние.

```dockerfile
# ПЛОХО — 3 лишних слоя
RUN apk add --no-cache cronie
RUN apk add --no-cache bash
RUN apk add --no-cache postgresql-client

# ХОРОШО — 1 слой
RUN apk add --no-cache cronie bash postgresql-client
```

Для apt то же самое, плюс нужно чистить кеш в том же слое:

```dockerfile
# ПЛОХО — кеш apt остаётся в слое
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ХОРОШО — обновление, установка и очистка в одном слое
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

Если `rm` сделать отдельным `RUN` — файлы удалятся в новом слое, но в предыдущем они всё ещё есть, и образ не уменьшится.

---

## Что делает `--no-cache` у apk?

Флаг `--no-cache` говорит Alpine не сохранять локальный индекс пакетов (`/var/cache/apk/`). Без него Alpine скачивает индекс и оставляет его в слое — это лишние мегабайты, которые в контейнере не нужны.

```dockerfile
# Без --no-cache пришлось бы чистить вручную
RUN apk update && apk add postgresql-client curl && rm -rf /var/cache/apk/*

# --no-cache делает то же самое одной командой
RUN apk add --no-cache postgresql-client curl
```

Аналог для apt — `--no-install-recommends` + `rm -rf /var/lib/apt/lists/*`.

---

## Что такое cache mount и чем он отличается от кеша слоёв?

Кеш слоёв Docker — это "всё или ничего". Если `package.json` изменился (добавили одну зависимость), слой с `pnpm install` полностью инвалидируется, и все пакеты скачиваются заново.

`--mount=type=cache` монтирует персистентную директорию хоста в контейнер на время сборки. Пакетный менеджер хранит свой store в этой директории, и она переживает инвалидацию слоёв.

```dockerfile
# pnpm — store сохраняется между сборками
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile

# npm — кеш скачанных тарболов сохраняется
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# pip
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

Даже если слой пересобирается, pnpm/npm видит, что большинство пакетов уже есть в store, и скачивает только новые. Вместо полной переустановки — доустановка того, что добавилось.

Требует BuildKit (`DOCKER_BUILDKIT=1` или Docker версии 23+).

---

## Зачем удалять devDependencies после сборки?

В builder-стадии нужны devDependencies (TypeScript, ESLint, тестовые фреймворки) для сборки. Но в финальный образ они попадать не должны. `pnpm prune --prod` удаляет всё, кроме production-зависимостей:

```dockerfile
# В builder-стадии, после билда
RUN pnpm build
RUN pnpm prune --prod --ignore-scripts
```

Это уменьшает `node_modules`, который копируется в runner-стадию. Для npm аналог — `npm prune --production`.

---

## Почему нельзя запускать контейнер от root?

Если злоумышленник получает RCE (Remote Code Execution) внутрь контейнера, он получает root-привилегии. При неправильной конфигурации (privileged-режим, монтирование docker.sock) root в контейнере фактически равен root на хосте.

В образах `node:*-alpine` уже есть пользователь `node`. Достаточно переключиться на него в runner-стадии:

```dockerfile
# В runner-стадии, после COPY
RUN chown -R node:node /app
USER node
CMD ["node", "dist/main.js"]
```

Для других образов можно создать пользователя:

```dockerfile
RUN addgroup --system app && adduser --system --ingroup app app
USER app
```

---

## Зачем добавлять HEALTHCHECK?

Без `HEALTHCHECK` Docker (и оркестраторы вроде Swarm) не знает, жив ли процесс внутри контейнера. Контейнер может висеть с зависшим процессом, но Docker будет считать его здоровым, потому что PID 1 существует.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

Если health check падает 3 раза подряд — Docker помечает контейнер как `unhealthy`, и оркестратор может его перезапустить. В Kubernetes обычно используются `livenessProbe` / `readinessProbe` вместо этого.

---

## Как правильно передавать секреты при сборке?

Секреты (API-ключи, токены, пароли) нельзя передавать через `COPY` или `ENV` — они остаются в слоях образа навсегда, даже если потом удалить файл.

```dockerfile
# ПЛОХО — .env остаётся в слое, даже если потом RUN rm .env
COPY .env .
RUN echo $SECRET_KEY

# ХОРОШО — BuildKit secrets не попадают в слой
RUN --mount=type=secret,id=db_password \
    cat /run/secrets/db_password | setup_db
```

```bash
docker build --secret id=db_password,src=./password.txt .
```

Секрет монтируется как временный файл только на время выполнения `RUN` и не сохраняется ни в одном слое.

---

## В чём разница между ENTRYPOINT и CMD?

`CMD` — команда по умолчанию, которую легко переопределить при `docker run`. `ENTRYPOINT` — фиксированная точка входа, `docker run` добавляет аргументы к ней.

```dockerfile
# CMD — легко переопределить
CMD ["node", "dist/main.js"]
# docker run myimage node other-script.js  → запустит other-script.js

# ENTRYPOINT — фиксированная команда
ENTRYPOINT ["node", "dist/main.js"]
# docker run myimage --port=3000  → запустит node dist/main.js --port=3000
```

Антипаттерн — `ENTRYPOINT ["sh", "-c", "pnpm start:prod"]`. Проблемы: `sh -c` не передаёт сигналы (SIGTERM) дочернему процессу, и контейнер не останавливается gracefully. Лучше использовать exec-форму:

```dockerfile
CMD ["node", "dist/main.js"]
```

---

## Как сканировать образ на уязвимости?

```bash
docker scout cves myimage:latest      # Docker Scout (встроен в Docker Desktop)
trivy image myimage:latest             # Trivy (open source)
```

Сканеры проверяют установленные пакеты (apk, apt, npm) и сообщают об известных CVE. Имеет смысл встраивать в CI — чтобы образ с критической уязвимостью не попал в production.

---

## Чеклист оптимизации

1. Использовать `slim` / `alpine` / `distroless` базовый образ
2. Пиннить версию базового образа (не `latest`)
3. Multi-stage build — отделить сборку от runtime
4. `.dockerignore` — исключить .git, node_modules, .env
5. Порядок COPY — сначала lock-файл и манифест, потом код
6. `--frozen-lockfile` / `npm ci` — детерминистичная установка
7. `--mount=type=cache` — не качать пакеты заново при изменении зависимостей
8. Объединять `RUN` + чистить кеш в одном слое
9. `--no-cache` для apk, `--no-cache-dir` для pip
10. `pnpm prune --prod` — удалить devDependencies после сборки
11. `USER node` — не запускать от root
12. Не хранить секреты в образе — использовать `--mount=type=secret`
13. `HEALTHCHECK` для production
14. Использовать `CMD` в exec-форме, избегать `sh -c`
