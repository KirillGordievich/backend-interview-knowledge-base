# Docker — Оптимизация образов

## Почему размер образа важен

- Быстрее pull/push → быстрее деплой
- Меньше attack surface (меньше пакетов = меньше CVE)
- Меньше потребление диска и сети
- Быстрее автоскейлинг (новый под = pull образа)

---

## Выбор базового образа

| Образ | Размер | Когда использовать |
|---|---|---|
| `python:3.12` | ~900 MB | Нужны системные пакеты, C-расширения |
| `python:3.12-slim` | ~150 MB | Большинство приложений |
| `python:3.12-alpine` | ~50 MB | Минимальный размер, но musl вместо glibc |
| `node:20` | ~1 GB | Нужны нативные модули |
| `node:20-alpine` | ~130 MB | Большинство приложений |
| `distroless` | ~20 MB | Только runtime, без shell/пакетного менеджера |

**Alpine:** основан на musl libc. Некоторые Python-пакеты (numpy, pandas) не имеют бинарных wheel для musl → компилируются из исходников → медленная сборка, больше образ. Для Python обычно лучше `slim`.

---

## Multi-stage builds

Разделение сборки и runtime → в финальный образ попадает только результат:

```dockerfile
# === Stage 1: сборка ===
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# === Stage 2: runtime ===
FROM python:3.12-slim

WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .

USER nobody
CMD ["uvicorn", "main:app", "--host", "0.0.0.0"]
```

```dockerfile
# Пример для Go — финальный образ ~10 MB
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM scratch
COPY --from=builder /app/server /server
CMD ["/server"]
```

```dockerfile
# Пример для Node.js
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json .
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
CMD ["node", "dist/main.js"]
```

---

## Оптимизация слоёв и кэша

### Порядок инструкций

Docker кэширует слои. Если слой изменился — все последующие пересобираются.

```dockerfile
# ПЛОХО — любое изменение кода инвалидирует кэш зависимостей
COPY . .
RUN pip install -r requirements.txt

# ХОРОШО — зависимости кэшируются отдельно
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### Объединение RUN

```dockerfile
# ПЛОХО — 3 слоя, apt cache остаётся
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ХОРОШО — 1 слой, чисто
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

### Кэш pip/npm

```dockerfile
# Не сохранять кэш pip
RUN pip install --no-cache-dir -r requirements.txt

# Или использовать BuildKit cache mount (не попадает в слой)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

---

## Безопасность

### Не запускать от root

```dockerfile
# Создать пользователя
RUN addgroup --system app && adduser --system --ingroup app app
USER app

# Или использовать встроенного
USER nobody
```

### Не хранить секреты в образе

```dockerfile
# ПЛОХО — секрет остаётся в слое навсегда
COPY .env .
RUN echo $SECRET_KEY

# ХОРОШО — BuildKit secrets (не попадают в слой)
RUN --mount=type=secret,id=db_password \
    cat /run/secrets/db_password | setup_db
```

```bash
docker build --secret id=db_password,src=./password.txt .
```

### HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

---

## Сканирование уязвимостей

```bash
docker scout cves myimage:latest      # Docker Scout (встроен в Docker Desktop)
trivy image myimage:latest             # Trivy (open source)
```

---

## Чеклист оптимизации

1. Использовать `slim` / `alpine` / `distroless` базовый образ
2. Multi-stage build — отделить сборку от runtime
3. `.dockerignore` — исключить .git, node_modules, .env, __pycache__
4. Порядок COPY — сначала зависимости, потом код
5. Объединять RUN + очищать кэш в одном слое
6. `--no-cache-dir` для pip, `npm ci` вместо `npm install`
7. `USER` — не запускать от root
8. Не хранить секреты в образе
9. HEALTHCHECK для production

---

## Вопросы на собеседовании

**Что такое multi-stage build и зачем он нужен?**
Dockerfile с несколькими `FROM`. Позволяет использовать один образ для сборки (компилятор, dev-зависимости), а в финальный образ копировать только артефакты. Результат — маленький и безопасный production-образ.

**Почему порядок инструкций в Dockerfile важен?**
Docker кэширует слои. При изменении слоя все последующие пересобираются. Если COPY кода идёт до установки зависимостей, любое изменение кода инвалидирует кэш зависимостей → медленная пересборка.

**Чем `alpine` плох для Python?**
Alpine использует musl libc вместо glibc. Многие Python-пакеты (numpy, pandas, psycopg2) не имеют pre-built wheels для musl → собираются из C-исходников → долгая сборка, нужны dev-пакеты, образ может стать больше, чем `slim`.

**Как уменьшить размер образа?**
Slim/distroless база, multi-stage build, `.dockerignore`, объединение RUN, очистка кэша пакетных менеджеров, `--no-install-recommends` для apt.

**Почему нельзя запускать контейнер от root?**
Если злоумышленник получает доступ внутрь контейнера (RCE), он получает root. При неправильной конфигурации (привилегированный режим, монтирование docker.sock) root в контейнере = root на хосте.
