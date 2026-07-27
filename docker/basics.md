# Docker — Основы

## Что такое Docker

**Docker** — платформа для контейнеризации приложений. Позволяет упаковать приложение со всеми зависимостями в изолированный контейнер, который одинаково работает на любом окружении.

**Контейнер vs VM:**

| | Контейнер | Виртуальная машина |
|---|---|---|
| Изоляция | Уровень процессов (namespaces, cgroups) | Полная (гипервизор) |
| Ядро | Общее с хостом | Своё |
| Размер | МБ | ГБ |
| Запуск | Секунды | Минуты |
| Overhead | Минимальный | Значительный |

---

## Архитектура Docker

Docker использует **клиент-серверную** архитектуру:

- **Docker CLI** (клиент) — отправляет команды через REST API
- **Docker Daemon** (`dockerd`) — управляет образами, контейнерами, сетями, томами
- **containerd** — среда выполнения контейнеров (container runtime)
- **runc** — низкоуровневый OCI runtime, непосредственно создаёт контейнер через системные вызовы

```
CLI → Docker Daemon → containerd → runc → контейнер
```

---

## Image (образ)

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

**Тег** — версия образа. `nginx:1.25`, `python:3.12-slim`. Тег `latest` — не «последний», а дефолтный (может быть устаревшим).

---

## Container (контейнер)

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

**Флаги `docker run`:**

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

## Dockerfile

Текстовый файл с инструкциями для сборки образа.

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

### ENTRYPOINT vs CMD

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

### exec-форма vs shell-форма

```dockerfile
# exec-форма (рекомендуется) — PID 1, получает сигналы
CMD ["python", "app.py"]

# shell-форма — запускается через /bin/sh -c, PID 1 = sh
CMD python app.py
```

Shell-форма не получает SIGTERM напрямую → контейнер убивается через 10 секунд по SIGKILL. Всегда используй exec-форму.

---

## .dockerignore

Исключает файлы из контекста сборки (аналог `.gitignore`):

```
.git
.env
__pycache__
node_modules
*.pyc
.venv
```

Уменьшает размер контекста → ускоряет сборку, предотвращает утечку секретов.

---

## Жизненный цикл контейнера

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

## Изоляция: namespaces и cgroups

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

## Docker Registry

**Registry** — хранилище образов.

- **Docker Hub** — публичный реестр по умолчанию
- **Приватные:** AWS ECR, GCR, GitHub Container Registry, self-hosted

```bash
docker login
docker tag myapp:v1 registry.example.com/myapp:v1
docker push registry.example.com/myapp:v1
docker pull registry.example.com/myapp:v1
```

---

## Вопросы на собеседовании

**В чём разница между контейнером и виртуальной машиной?**
Контейнер разделяет ядро с хостом и изолирует процессы через namespaces/cgroups. VM запускает полноценную ОС через гипервизор. Контейнеры легче, быстрее запускаются, но изоляция менее строгая.

**Что происходит при `docker run nginx`?**
1. Docker ищет образ локально, если нет — pull из registry
2. Создаёт writable layer поверх образа
3. Создаёт namespaces и cgroups
4. Настраивает сеть (виртуальный интерфейс, bridge)
5. Запускает процесс из CMD/ENTRYPOINT

**Почему нельзя использовать shell-форму CMD?**
Shell-форма запускает процесс через `/bin/sh -c`, и PID 1 становится `sh`, а не приложение. `sh` не пробрасывает сигналы → при `docker stop` приложение не получает SIGTERM и убивается через 10 секунд SIGKILL-ом.

**Что такое dangling images?**
Образы без тега (`<none>:<none>`), обычно оставшиеся после пересборки. Удаляются через `docker image prune`.

**Зачем нужен `.dockerignore`?**
Исключает файлы из контекста сборки → ускоряет сборку, уменьшает размер образа, предотвращает попадание секретов (.env, .git) в образ.
