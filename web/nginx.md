# Nginx

## Что такое Nginx

Nginx — высокопроизводительный веб-сервер и reverse proxy. Работает на event-driven, asynchronous архитектуре — один воркер обслуживает тысячи соединений без создания нового потока на каждое.

**Роли nginx:**
- **Веб-сервер** — отдача статических файлов (HTML, CSS, JS, изображения)
- **Reverse proxy** — принимает запросы снаружи, передаёт на backend (Uvicorn, Gunicorn, Node.js)
- **Load balancer** — распределяет запросы между несколькими backend-серверами
- **SSL terminator** — обрабатывает HTTPS, на backend передаёт HTTP
- **Кэш** — кэширует ответы backend

---

## Структура конфигурации

```
nginx.conf
├── main context        (глобальные настройки)
├── events {}           (настройки соединений)
└── http {}
    ├── upstream {}     (группа backend-серверов)
    └── server {}       (виртуальный хост)
        └── location {} (обработка URL)
```

```nginx
# /etc/nginx/nginx.conf

worker_processes auto;          # число воркеров = число CPU ядер

events {
    worker_connections 1024;    # макс. соединений на воркер
    use epoll;                  # Linux: самый быстрый механизм I/O
}

http {
    include mime.types;
    sendfile on;                # zero-copy передача файлов
    tcp_nopush on;              # отправлять заголовки и начало файла в одном пакете
    gzip on;                    # сжатие ответов
    keepalive_timeout 65;       # держать соединение открытым

    include /etc/nginx/conf.d/*.conf;
}
```

---

## Server block (виртуальный хост)

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    # Редирект HTTP → HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/ssl/example.com.crt;
    ssl_certificate_key /etc/ssl/example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    root /var/www/html;
    index index.html;

    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log;
}
```

---

## Location block

```nginx
server {
    # Точное совпадение (приоритет выше)
    location = /health {
        return 200 'OK';
        add_header Content-Type text/plain;
    }

    # Prefix match
    location /static/ {
        root /var/www;               # файлы из /var/www/static/
        expires 30d;                 # кэширование на стороне клиента
        add_header Cache-Control "public";
    }

    # Regex match (~* — без учёта регистра)
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 7d;
    }

    # Все остальные → backend
    location / {
        proxy_pass http://backend;
    }
}
```

**Приоритет location:** `=` > `^~` (prefix, stop search) > `~` regex > prefix > `/`

---

## Reverse Proxy

```nginx
upstream backend {
    server 127.0.0.1:8000;    # Uvicorn / Gunicorn
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;

        # Передать реальный IP клиента на backend
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Таймауты
        proxy_connect_timeout 5s;
        proxy_read_timeout    60s;
        proxy_send_timeout    60s;

        # Буферизация
        proxy_buffering    on;
        proxy_buffer_size  8k;
    }
}
```

**FastAPI / Uvicorn:**
```bash
# Запустить Uvicorn на localhost
uvicorn app.main:app --host 127.0.0.1 --port 8000 --workers 4
```

---

## Load Balancing

```nginx
upstream backend {
    # Методы балансировки:

    # round-robin (по умолчанию) — запросы по кругу
    server 10.0.0.1:8000;
    server 10.0.0.2:8000;
    server 10.0.0.3:8000;

    # least_conn — на сервер с наименьшим числом активных соединений
    least_conn;

    # ip_hash — один клиент всегда на один сервер (sticky sessions без cookie)
    ip_hash;

    # Веса
    server 10.0.0.1:8000 weight=3;   # получит в 3 раза больше запросов
    server 10.0.0.2:8000 weight=1;

    # Резервный сервер (используется если все основные недоступны)
    server 10.0.0.3:8000 backup;

    # Параметры health check
    server 10.0.0.1:8000 max_fails=3 fail_timeout=30s;

    # Keep-alive соединения к backend
    keepalive 32;
}
```

---

## WebSocket Proxy

```nginx
server {
    location /ws/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;

        # Необходимо для Upgrade handshake
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_read_timeout 3600s;   # долгое соединение
    }
}
```

---

## Кэширование

```nginx
http {
    # Зона кэша: 10MB метаданных, 100MB данных, TTL 60 мин
    proxy_cache_path /var/cache/nginx
        levels=1:2
        keys_zone=my_cache:10m
        max_size=100m
        inactive=60m;

    server {
        location /api/ {
            proxy_pass http://backend;
            proxy_cache my_cache;
            proxy_cache_valid 200 10m;    # кэшировать 200 на 10 минут
            proxy_cache_valid 404 1m;
            proxy_cache_use_stale error timeout;  # отдать старый кэш при ошибке

            add_header X-Cache-Status $upstream_cache_status;  # HIT / MISS
        }
    }
}
```

---

## Ограничение запросов (Rate Limiting)

```nginx
http {
    # Зона: ключ = IP, 10MB памяти, лимит 10 req/sec
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            # burst=20: до 20 запросов в очереди
            # nodelay: не задерживать burst-запросы
            proxy_pass http://backend;
        }
    }
}
```

---

## Gzip

```nginx
http {
    gzip on;
    gzip_vary on;           # добавить Vary: Accept-Encoding
    gzip_min_length 1024;   # не сжимать меньше 1KB
    gzip_comp_level 6;      # уровень сжатия (1-9, 6 — баланс)
    gzip_types
        text/plain
        text/css
        application/json
        application/javascript
        application/xml
        image/svg+xml;
}
```

---

## Полезные команды

```bash
nginx -t                    # проверить конфигурацию на ошибки
nginx -s reload             # перезагрузить конфиг без остановки (SIGHUP)
nginx -s stop               # остановить немедленно
nginx -s quit               # остановить gracefully

systemctl status nginx
systemctl reload nginx      # то же что nginx -s reload

# Логи
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

---

## Типичная конфигурация для Python backend

```nginx
upstream fastapi {
    server 127.0.0.1:8000;
    keepalive 32;
}

server {
    listen 80;
    server_name api.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    client_max_body_size 10M;

    location /static/ {
        alias /var/www/static/;
        expires 30d;
    }

    location / {
        proxy_pass http://fastapi;
        proxy_http_version 1.1;
        proxy_set_header Connection "";        # для keepalive к backend
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
