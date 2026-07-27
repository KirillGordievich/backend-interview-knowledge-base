# Docker — Тома и хранение данных

## Проблема

Контейнер имеет writable layer (Copy-on-Write), но:
- Данные теряются при удалении контейнера
- Writable layer привязан к конкретному хосту
- Запись через CoW медленнее, чем напрямую в файловую систему

Для персистентного хранения Docker предоставляет три механизма.

---

## Типы монтирования

| Тип | Управление | Расположение | Назначение |
|---|---|---|---|
| **Volume** | Docker | `/var/lib/docker/volumes/` | Персистентные данные (БД, файлы) |
| **Bind mount** | Пользователь | Любой путь на хосте | Разработка, конфиги |
| **tmpfs** | Память | RAM | Временные данные, секреты |

---

## Volumes (тома)

**Volume** — предпочтительный способ хранения данных. Управляется Docker, не зависит от структуры файловой системы хоста.

```bash
# Создание и управление
docker volume create pgdata
docker volume ls
docker volume inspect pgdata
docker volume rm pgdata
docker volume prune                   # удалить неиспользуемые

# Использование
docker run -d \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Альтернативный синтаксис (--mount)
docker run -d \
  --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
  postgres:16
```

**Преимущества volumes:**
- Безопасное хранение вне контейнера
- Работают на Linux и Windows
- Могут использовать драйверы (NFS, облачное хранилище)
- Можно делать backup через промежуточный контейнер
- Безопаснее — нельзя случайно смонтировать системную директорию

---

## Bind mounts

Монтирование директории/файла хоста в контейнер. Путь на хосте должен существовать.

```bash
# Разработка: код с хоста → контейнер (live reload)
docker run -d \
  -v $(pwd)/src:/app/src \
  myapp

# Read-only
docker run -d \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx

# --mount синтаксис (строже, не создаёт путь автоматически)
docker run -d \
  --mount type=bind,source=$(pwd)/src,target=/app/src \
  myapp
```

**Bind mount vs Volume:**

| | Volume | Bind mount |
|---|---|---|
| Управление | Docker | Пользователь |
| Путь | Абстрактное имя | Абсолютный путь на хосте |
| Production | Да | Нет (зависит от хоста) |
| Разработка | Нет | Да (live reload) |
| Безопасность | Изолировано | Доступ к файлам хоста |

---

## tmpfs

Данные хранятся в оперативной памяти, не записываются на диск:

```bash
docker run -d \
  --tmpfs /tmp \
  myapp

docker run -d \
  --mount type=tmpfs,target=/tmp,tmpfs-size=100m \
  myapp
```

Применение: временные файлы, кэш, секреты (не попадут на диск).

---

## Volumes в Docker Compose

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data      # named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # bind mount

  web:
    build: .
    volumes:
      - ./src:/app/src                       # bind mount (разработка)
      - node_modules:/app/node_modules       # named volume (не перезаписывается хостом)

volumes:
  pgdata:                                    # объявление named volume
  node_modules:
```

### Anonymous volumes

```yaml
services:
  web:
    volumes:
      - /app/node_modules   # anonymous volume — сохраняет содержимое из образа
```

Трюк: при bind mount `./:/app` хостовые файлы перезаписывают содержимое контейнера. Anonymous volume на `/app/node_modules` сохраняет `node_modules` из образа, не перезаписывая хостовым (пустым) каталогом.

---

## Backup и восстановление

```bash
# Backup тома через промежуточный контейнер
docker run --rm \
  -v pgdata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/pgdata.tar.gz -C /data .

# Restore
docker run --rm \
  -v pgdata:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/pgdata.tar.gz -C /data
```

---

## Вопросы на собеседовании

**Чем volume отличается от bind mount?**
Volume управляется Docker, хранится в `/var/lib/docker/volumes/`, не зависит от файловой системы хоста. Bind mount — прямое монтирование пути хоста. Volumes безопаснее и предпочтительнее в production.

**Что произойдёт с данными при `docker compose down`?**
Контейнеры и сети удаляются, но named volumes сохраняются. Для удаления томов нужен флаг `-v`: `docker compose down -v`.

**Зачем anonymous volume для node_modules?**
При bind mount `./:/app` содержимое хоста перезаписывает всё в `/app`, включая `node_modules` из образа. Anonymous volume на `/app/node_modules` «защищает» установленные зависимости от перезаписи пустой директорией хоста.

**Как данные PostgreSQL переживают пересоздание контейнера?**
Данные хранятся в named volume, примонтированном к `/var/lib/postgresql/data`. При `docker compose down && docker compose up` создаётся новый контейнер, но он подключает тот же том с данными.
