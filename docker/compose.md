# Docker — Docker Compose

## Что такое Docker Compose

**Docker Compose** — инструмент для определения и запуска многоконтейнерных приложений. Описываешь сервисы, сети и тома в YAML-файле → запускаешь одной командой.

```yaml
# compose.yaml (или docker-compose.yml)
services:
  web:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - db
      - redis
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
    volumes:
      - .:/app

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

---

## Команды

```bash
docker compose up                  # запустить все сервисы
docker compose up -d               # в фоне (detached)
docker compose up --build          # пересобрать образы перед запуском
docker compose down                # остановить и удалить контейнеры, сети
docker compose down -v             # + удалить тома
docker compose ps                  # статус сервисов
docker compose logs web            # логи конкретного сервиса
docker compose logs -f             # follow всех логов
docker compose exec web bash       # shell внутри запущенного сервиса
docker compose run web pytest      # одноразовый запуск команды
docker compose build               # только сборка образов
docker compose pull                # скачать образы
docker compose restart web         # перезапустить сервис
docker compose stop                # остановить без удаления
```

---

## Ключевые директивы

### services

```yaml
services:
  app:
    image: python:3.12              # готовый образ
    # ИЛИ
    build:                          # сборка из Dockerfile
      context: .
      dockerfile: Dockerfile.prod
      args:
        BUILD_ENV: production

    ports:
      - "8000:8000"                 # хост:контейнер
    expose:
      - "8000"                      # только внутри сети (без проброса на хост)

    environment:                    # переменные окружения
      - DEBUG=false
    env_file:                       # из файла
      - .env

    volumes:
      - ./src:/app/src              # bind mount
      - data:/app/data              # named volume

    command: ["uvicorn", "main:app"]  # переопределить CMD
    entrypoint: ["/entrypoint.sh"]    # переопределить ENTRYPOINT

    restart: unless-stopped         # политика перезапуска

    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
```

### depends_on

Управляет порядком запуска (но не ждёт готовности!):

```yaml
services:
  web:
    depends_on:
      db:
        condition: service_healthy   # ждать healthcheck
      redis:
        condition: service_started   # просто запуск (по умолчанию)

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5
```

**condition:**
- `service_started` — контейнер запущен
- `service_healthy` — healthcheck пройден
- `service_completed_successfully` — контейнер завершился с кодом 0

### networks

```yaml
services:
  web:
    networks:
      - frontend
      - backend
  db:
    networks:
      - backend

networks:
  frontend:
  backend:
```

Сервисы в одной сети видят друг друга по имени сервиса (DNS). Сервисы в разных сетях изолированы.

### volumes

```yaml
volumes:
  pgdata:                           # named volume (управляется Docker)
    driver: local
  cache:
    external: true                  # том создан заранее, не удаляется при down -v
```

---

## Переменные и интерполяция

```yaml
services:
  db:
    image: postgres:${POSTGRES_VERSION:-16}   # значение по умолчанию
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}       # из .env или окружения
```

```bash
# .env (автоматически подтягивается из текущей директории)
POSTGRES_VERSION=16
DB_PASSWORD=secret
```

---

## Профили (profiles)

Запускают сервисы выборочно:

```yaml
services:
  web:
    build: .

  debug:
    image: busybox
    profiles:
      - debug

  test:
    build: .
    command: pytest
    profiles:
      - test
```

```bash
docker compose up                        # только web (без профиля)
docker compose --profile debug up        # web + debug
docker compose --profile test run test   # запустить тесты
```

---

## Множественные compose-файлы

```bash
# Override: второй файл перезаписывает/дополняет первый
docker compose -f compose.yaml -f compose.prod.yaml up
```

```yaml
# compose.yaml — базовый
services:
  web:
    build: .
    ports:
      - "8000:8000"

# compose.prod.yaml — override для production
services:
  web:
    environment:
      - DEBUG=false
    restart: always
```

---

## Вопросы на собеседовании

**Чем `depends_on` отличается от ожидания готовности?**
`depends_on` управляет порядком запуска контейнеров, но не ждёт, пока сервис реально станет доступен. Для ожидания готовности используют `condition: service_healthy` с `healthcheck`, либо скрипты типа wait-for-it.

**Как работает DNS в Compose?**
Compose создаёт bridge-сеть для проекта. Каждый сервис доступен по имени из `services:` как hostname. `web` может обратиться к `db` просто по имени `db:5432`.

**Разница между `ports` и `expose`?**
`ports` пробрасывает порт на хост (доступен извне). `expose` открывает порт только внутри Docker-сети (для межсервисного общения).

**Как передавать секреты в Compose?**
Через `env_file` (`.env` файл не коммитится в git), Docker secrets (в Swarm mode), или внешние хранилища (Vault). Никогда не хардкодить в compose-файле.
