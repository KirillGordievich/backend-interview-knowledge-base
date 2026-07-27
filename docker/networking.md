# Docker — Сети

## Сетевые драйверы

Docker предоставляет несколько сетевых драйверов для разных сценариев:

| Драйвер | Описание | Применение |
|---|---|---|
| `bridge` | Изолированная сеть на хосте (по умолчанию) | Контейнеры на одном хосте |
| `host` | Контейнер использует сетевой стек хоста | Максимальная производительность, нет изоляции |
| `none` | Без сети | Полная изоляция |
| `overlay` | Сеть между несколькими Docker-хостами | Docker Swarm, кластеры |
| `macvlan` | Контейнер получает MAC-адрес в физической сети | Legacy-интеграция |

---

## Bridge (по умолчанию)

При установке Docker создаёт сеть `bridge` (интерфейс `docker0`).

```bash
# Default bridge
docker run -d --name app1 nginx
docker run -d --name app2 nginx
# app1 и app2 в одной сети, но НЕ видят друг друга по имени
# (default bridge не поддерживает DNS)
```

### User-defined bridge (рекомендуется)

```bash
docker network create mynet
docker run -d --name app1 --network mynet nginx
docker run -d --name app2 --network mynet nginx
# app1 может обратиться к app2 по имени: curl http://app2:80
```

**Преимущества user-defined bridge перед default:**

| Default bridge | User-defined bridge |
|---|---|
| Нет DNS (только IP или `--link`) | Автоматический DNS по имени контейнера |
| Все контейнеры в одной сети | Изоляция между сетями |
| Нельзя подключить/отключить на лету | `docker network connect/disconnect` |

---

## Host

Контейнер разделяет сетевой стек хоста — нет NAT, нет проброса портов:

```bash
docker run -d --network host nginx
# nginx доступен на порту 80 хоста напрямую
# -p не нужен и игнорируется
```

Используют когда NAT даёт заметный overhead (высоконагруженные сервисы). Работает только на Linux.

---

## None

```bash
docker run -d --network none alpine
# Контейнер полностью изолирован от сети
# Есть только loopback (lo)
```

---

## Команды

```bash
docker network ls                           # список сетей
docker network create mynet                 # создать bridge-сеть
docker network create --driver overlay mynet # overlay-сеть
docker network inspect mynet                # детали (подсети, контейнеры)
docker network connect mynet container1     # подключить контейнер к сети
docker network disconnect mynet container1  # отключить
docker network rm mynet                     # удалить сеть
docker network prune                        # удалить неиспользуемые
```

---

## Проброс портов

```bash
docker run -p 8080:80 nginx          # хост 8080 → контейнер 80
docker run -p 127.0.0.1:8080:80 nginx # только localhost
docker run -p 8080:80/udp nginx       # UDP
docker run -P nginx                   # рандомные порты для всех EXPOSE
```

**Как работает:** Docker добавляет правила iptables (DNAT) для маршрутизации трафика с хоста в контейнер через bridge.

---

## DNS в Docker

В user-defined сетях Docker запускает встроенный DNS-сервер (`127.0.0.11`):
- Контейнеры резолвятся по имени (`app1`, `db`)
- В Compose — по имени сервиса
- Для внешних доменов — проксирует на DNS хоста

```bash
# Внутри контейнера
nslookup db          # → IP контейнера db
cat /etc/resolv.conf # nameserver 127.0.0.11
```

---

## Сеть в Docker Compose

Compose автоматически создаёт bridge-сеть `<project>_default`:

```yaml
services:
  web:
    build: .
    # доступен как "web" внутри сети
  db:
    image: postgres:16
    # доступен как "db" внутри сети
```

```python
# web может обратиться к db по имени
DATABASE_URL = "postgresql://user:pass@db:5432/mydb"
```

---

## Вопросы на собеседовании

**Почему контейнеры не видят друг друга по имени в default bridge?**
Default bridge не запускает встроенный DNS-сервер. Для обнаружения по имени нужно использовать user-defined bridge-сеть, где Docker предоставляет автоматический DNS.

**Что происходит при `-p 8080:80`?**
Docker создаёт правило iptables (DNAT), которое перенаправляет трафик с порта 8080 хоста на порт 80 контейнера через bridge-интерфейс. Контейнер получает запрос на свой виртуальный интерфейс `eth0`.

**Когда использовать `--network host`?**
Когда NAT создаёт значительный overhead (высокий RPS, latency-sensitive приложения). Работает только на Linux. Теряется сетевая изоляция — контейнер видит все порты хоста.

**Как контейнер в одной сети общается с контейнером в другой?**
Напрямую — никак, сети изолированы. Нужно либо подключить контейнер к обеим сетям (`docker network connect`), либо пробросить порт на хост.
