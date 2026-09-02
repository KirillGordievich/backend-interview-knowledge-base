# Docker — Основы

## Что такое Docker и зачем он нужен

**Docker** — платформа для контейнеризации приложений. Позволяет упаковать приложение со всеми зависимостями в изолированный контейнер, который одинаково работает на любом окружении.

**Чем контейнер отличается от виртуальной машины?**

Контейнер работает поверх ядра хоста и изолирует только процессы, а VM запускает полноценную гостевую ОС через гипервизор. Из-за этого контейнеры значительно легче и быстрее:

| | Контейнер | Виртуальная машина |
|---|---|---|
| Изоляция | Уровень процессов (namespaces, cgroups) | Полная (гипервизор) |
| Ядро | Общее с хостом | Своё |
| Размер | МБ | ГБ |
| Запуск | Секунды | Минуты |
| Overhead | Минимальный | Значительный |

---

## Как устроена архитектура Docker

Docker использует **клиент-серверную** архитектуру:

- **Docker CLI** (клиент) — отправляет команды через REST API (По умолчанию CLI обычно общается с daemon'ом через Unix socket, а не HTTP)
- **Docker Daemon** (`dockerd`) — управляет образами, контейнерами, сетями, томами
- **containerd** — среда выполнения контейнеров (container runtime)
- **runc** — низкоуровневый OCI runtime, непосредственно создаёт контейнер через системные вызовы

```
CLI → Docker Daemon → containerd → runc → контейнер
```

---

## Что такое Docker Image (образ)

**Образ** — read-only шаблон с файловой системой и метаданными для создания контейнеров.

**Слои (layers):** каждая инструкция в Dockerfile создаёт новый слой. Слои кэшируются и переиспользуются между образами (UnionFS / OverlayFS).

```bash
docker images                    # список локальных образов
docker pull nginx:1.25           # скачать образ
docker rmi nginx:1.25            # удалить образ
docker image prune               # удалить неиспользуемые образы
docker image inspect nginx       # метаданные образа
docker history nginx              # слои образа
```

**Что такое тег?** Тег — это метка версии образа. По умолчанию используется тег `latest`, но на практике лучше указывать конкретную версию: `nginx:1.25`, `python:3.12-slim`, `node:22-alpine`. Это гарантирует воспроизводимость сборки.

---

## Что такое контейнер и как им управлять

**Контейнер** — запущенный экземпляр образа. Имеет свой writable layer поверх read-only слоёв образа (Copy-on-Write).

```bash
docker run -d --name app nginx               # запустить в фоне
docker run -it python:3.12 bash              # интерактивный режим
docker run --rm alpine echo "hello"          # удалить после завершения

docker ps                                     # запущенные контейнеры
docker ps -a                                  # все (включая остановленные)
docker stop app                               # SIGTERM → SIGKILL (через 10s)
docker kill app                               # SIGKILL немедленно
docker rm app                                 # удалить контейнер
docker logs app                               # логи (stdout/stderr)
docker logs -f app                            # follow логов
docker exec -it app bash                      # выполнить команду внутри
docker inspect app                            # полная информация (JSON)
```

**Основные флаги `docker run`:**

| Флаг | Назначение |
|---|---|
| `-d` | Detached (фон) |
| `-it` | Interactive + TTY |
| `--rm` | Удалить после остановки |
| `-p 8080:80` | Проброс порта (хост:контейнер) |
| `-v /host:/cont` | Монтирование тома |
| `-e KEY=VAL` | Переменная окружения |
| `--name` | Имя контейнера |
| `--restart` | Политика перезапуска |
| `--memory 512m` | Лимит памяти |
| `--cpus 1.5` | Лимит CPU |

---

## Как написать Dockerfile

Dockerfile — это текстовый файл с инструкциями для сборки образа. Docker читает его сверху вниз и выполняет каждую инструкцию, формируя слои образа.

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Основные инструкции:**

| Инструкция | Назначение |
|---|---|
| `FROM` | Базовый образ |
| `WORKDIR` | Рабочая директория |
| `COPY` | Копирование файлов в образ |
| `ADD` | Как COPY, но с распаковкой tar и поддержкой URL |
| `RUN` | Выполнение команды при сборке (новый слой) |
| `CMD` | Команда по умолчанию при запуске контейнера |
| `ENTRYPOINT` | Фиксированная команда (CMD становится аргументами) |
| `ENV` | Переменная окружения |
| `ARG` | Аргумент сборки (доступен только при `docker build`) |
| `EXPOSE` | Документирует порт (не пробрасывает!) |
| `VOLUME` | Точка монтирования |
| `USER` | Пользователь для последующих команд |
| `HEALTHCHECK` | Проверка здоровья контейнера |

**Слои (layers):** Docker image состоит из набора неизменяемых filesystem layers. Каждый слой содержит изменения файловой системы относительно предыдущего слоя. Слои кэшируются и могут переиспользоваться между сборками и образами.

`RUN`, `COPY` и `ADD` обычно создают новый filesystem layer. Такие инструкции, как `ENV`, `WORKDIR`, `USER`, `CMD`, `EXPOSE`, в основном изменяют конфигурацию/metadata образа и не создают отдельный filesystem layer.

Например:

```dockerfile
FROM python:3.12-slim

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

Base image
    ↓
COPY requirements.txt    ← layer
    ↓
RUN pip install          ← layer
    ↓
COPY . .                 ← layer
    ↓
CMD                      ← image configuration

Главные преимущества layers:

- **Caching** — неизменившиеся слои переиспользуются при пересборке
- **Переиспользование** — общие слои делятся между разными образами
- **Copy-on-Write** — контейнер пишет только в свой writable layer, не трогая образ
- **Эффективная передача** — при push/pull передаются только изменившиеся слои

Поэтому важно правильно располагать инструкции Dockerfile. Например, зависимости лучше копировать и устанавливать до исходного кода:

```dockerfile
COPY package.json pnpm-lock.yaml ./
RUN pnpm install

COPY . .
```

Тогда изменение исходного кода не инвалидирует cache установки зависимостей.

### В чём разница между ENTRYPOINT и CMD

```dockerfile
# CMD — можно переопределить при docker run
CMD ["python", "app.py"]
# docker run myimage python test.py  → запустит test.py

# ENTRYPOINT — фиксированная команда, CMD = аргументы
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run myimage test.py  → python test.py
```

| | ENTRYPOINT | CMD |
|---|---|---|
| Переопределение | `--entrypoint` | Аргументы после image name |
| Назначение | «Что запускать» | «С какими аргументами по умолчанию» |

### Чем отличается exec-форма от shell-формы

```dockerfile
# exec-форма (рекомендуется) — PID 1, получает сигналы
CMD ["python", "app.py"]

# shell-форма — запускается через /bin/sh -c, PID 1 = sh
CMD python app.py
```

Shell-форма не получает SIGTERM напрямую → контейнер убивается через 10 секунд по SIGKILL. Всегда используй exec-форму.

---

## Зачем нужен .dockerignore

Файл `.dockerignore` работает как `.gitignore` — исключает файлы из контекста сборки. Без него Docker отправляет всё содержимое директории в daemon, включая `.git`, `node_modules` и секреты:

```
.git
.env
__pycache__
node_modules
*.pyc
.venv
```

Это уменьшает размер контекста, ускоряет сборку и предотвращает случайное попадание секретов в образ.

---

## Какой жизненный цикл у контейнера

```
docker create → Created
docker start  → Running
docker pause  → Paused
docker unpause → Running
docker stop   → Exited (graceful)
docker kill   → Exited (force)
docker rm     → Deleted
```

**Политики перезапуска (`--restart`):**

| Политика | Поведение |
|---|---|
| `no` | Не перезапускать (по умолчанию) |
| `on-failure[:N]` | При ненулевом exit code (макс N раз) |
| `always` | Всегда (даже при ручном stop → после restart daemon) |
| `unless-stopped` | Как always, но не после ручного stop |

---

## Как Docker изолирует контейнеры: namespaces и cgroups

**Namespaces** — изоляция того, что контейнер *видит*:

| Namespace | Изолирует |
|---|---|
| PID | Процессы (PID 1 внутри контейнера) |
| NET | Сетевой стек (свой IP, порты) |
| MNT | Файловая система |
| UTS | Hostname |
| IPC | Inter-process communication |
| USER | UID/GID маппинг |

**cgroups** — ограничение того, сколько ресурсов контейнер *потребляет*:
- Лимиты CPU, памяти, I/O, сети
- Учёт и контроль ресурсов

---

## Что такое Docker Registry

**Registry** — хранилище Docker-образов. Когда вы делаете `docker pull`, образ скачивается именно из registry.

- **Docker Hub** — публичный реестр по умолчанию
- **Приватные:** AWS ECR, GCR, GitHub Container Registry, self-hosted

```bash
docker login
docker tag myapp:v1 registry.example.com/myapp:v1
docker push registry.example.com/myapp:v1
docker pull registry.example.com/myapp:v1
```

---

## Частые вопросы на собеседовании

**Что происходит при `docker run nginx`?**
1. Docker ищет образ локально, если нет — pull из registry
2. Создаёт writable layer поверх образа
3. Создаёт namespaces и cgroups
4. Настраивает сеть (виртуальный интерфейс, bridge)
5. Запускает процесс из CMD/ENTRYPOINT

**Что такое dangling images?**
Это образы без тега (`<none>:<none>`), которые обычно остаются после пересборки образа с тем же тегом. Старый образ теряет тег и становится «висячим». Удалить их можно через `docker image prune`.
