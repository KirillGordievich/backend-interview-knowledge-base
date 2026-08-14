# Kubernetes — Конфигурация и Health Checks

## ConfigMap

**ConfigMap** — хранение конфигурации (не секретной) отдельно от образа.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
  nginx.conf: |
    server {
      listen 80;
      location / { proxy_pass http://app:8000; }
    }
```

### Использование ConfigMap

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0

      # Как переменные окружения
      envFrom:
        - configMapRef:
            name: app-config        # все ключи как env vars

      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST    # один конкретный ключ

      # Как файлы (volume)
      volumeMounts:
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf       # один файл, не директория
  volumes:
    - name: config
      configMap:
        name: app-config
```

**Обновление:** при монтировании как volume — обновляется автоматически (с задержкой ~1 мин). Как env var — нет (нужен перезапуск Pod).

---

## Secret

**Secret** — хранение чувствительных данных (пароли, токены, TLS-сертификаты). Данные в base64 (не шифрование!).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: cG9zdGdyZXM=           # echo -n "postgres" | base64
  password: c2VjcmV0MTIz           # echo -n "secret123" | base64
```

```bash
# Или через kubectl (base64 автоматически)
kubectl create secret generic db-credentials \
  --from-literal=username=postgres \
  --from-literal=password=secret123
```

### Использование Secret

```yaml
spec:
  containers:
    - name: app
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password

      # Как файлы
      volumeMounts:
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secrets
      secret:
        secretName: db-credentials
```

### Типы Secret

| Тип | Назначение |
|---|---|
| `Opaque` | Произвольные данные (по умолчанию) |
| `kubernetes.io/tls` | TLS-сертификат (tls.crt + tls.key) |
| `kubernetes.io/dockerconfigjson` | Credentials для Docker registry |
| `kubernetes.io/basic-auth` | Логин/пароль |

**Важно:** Secret в etcd хранится в base64, не зашифрованным (по умолчанию). Для шифрования нужно включить encryption at rest или использовать внешние хранилища (Vault, AWS Secrets Manager) через External Secrets Operator.

---

## Resource Requests и Limits

**Requests** — гарантированные ресурсы (scheduler учитывает при размещении Pod).
**Limits** — максимальные ресурсы (превышение → OOM kill для памяти, throttling для CPU).

```yaml
spec:
  containers:
    - name: app
      resources:
        requests:
          memory: "128Mi"     # гарантировано
          cpu: "250m"         # 0.25 ядра
        limits:
          memory: "256Mi"     # максимум (OOM kill при превышении)
          cpu: "500m"         # максимум (throttling)
```

**CPU:** `1000m` = 1 ядро. `250m` = 0.25 ядра.
**Memory:** `Mi` (мебибайт), `Gi` (гибибайт).

### QoS классы

Kubernetes назначает QoS класс Pod на основе requests/limits:

| QoS класс | Условие | Приоритет при OOM |
|---|---|---|
| **Guaranteed** | requests = limits для всех контейнеров | Последний убивается |
| **Burstable** | requests < limits (или limits не заданы) | Средний |
| **BestEffort** | Ни requests, ни limits не заданы | Первый убивается |

---

## Probes (проверки здоровья)

### Liveness Probe

Проверяет, жив ли контейнер. Если проверка не проходит → контейнер перезапускается.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 15     # ждать перед первой проверкой
  periodSeconds: 10            # интервал
  failureThreshold: 3          # сколько неудач до перезапуска
  timeoutSeconds: 3            # таймаут запроса
```

Применение: контейнер завис (deadlock), приложение перестало отвечать.

### Readiness Probe

Проверяет, готов ли контейнер принимать трафик. Если не проходит → Pod удаляется из endpoints Service (трафик не идёт).

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

Применение: приложение запускается, но ещё не подключилось к БД, не загрузило кэш.

### Startup Probe

Проверяет, запустилось ли приложение. Пока не пройдёт — liveness и readiness не выполняются. Для медленно стартующих приложений.

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8000
  failureThreshold: 30         # 30 * 10 = 5 минут на запуск
  periodSeconds: 10
```

### Методы проверки

```yaml
# HTTP GET (самый частый)
httpGet:
  path: /health
  port: 8000

# TCP-соединение (для БД, Redis)
tcpSocket:
  port: 5432

# Выполнение команды (exit code 0 = успех)
exec:
  command: ["pg_isready", "-U", "postgres"]
```

### Сводка

| Probe | Что проверяет | При неудаче |
|---|---|---|
| **Liveness** | Жив ли контейнер | Перезапуск контейнера |
| **Readiness** | Готов ли принимать трафик | Убирает из Service endpoints |
| **Startup** | Запустилось ли приложение | Блокирует liveness/readiness |

---

## Вопросы на собеседовании

**Чем ConfigMap отличается от Secret?**
ConfigMap — для не секретных данных (конфиги, URL). Secret — для чувствительных данных (пароли, токены). Secret хранится в base64 (не шифрование!), но может быть зашифрован через encryption at rest. Оба можно подключать как env vars или файлы.

**Зачем нужны и requests, и limits?**
Requests — гарантированные ресурсы, scheduler использует их при размещении Pod на ноду. Limits — потолок потребления. Без requests — Pod BestEffort, первый кандидат на OOM kill. Без limits — Pod может съесть все ресурсы ноды.

**В чём разница между liveness и readiness probe?**
Liveness — контейнер жив? Если нет → перезапуск контейнера. Readiness — контейнер готов к трафику? Если нет → убирается из Service endpoints (трафик не приходит), но контейнер не перезапускается. При старте: readiness не проходит → Pod не получает трафик, пока не будет готов.

**Почему Secret в base64 — это не безопасно?**
base64 — это кодирование, не шифрование. Любой с доступом к etcd или API может декодировать. Нужно: включить encryption at rest в etcd, использовать RBAC для ограничения доступа к Secret, или внешний secrets manager (Vault, AWS Secrets Manager).

**Зачем нужен startup probe?**
Для приложений с долгим запуском (загрузка ML-модели, прогрев кэша). Без startup probe liveness probe начнёт проверять сразу после `initialDelaySeconds` и может перезапустить контейнер, который ещё не успел стартовать. startup probe блокирует остальные probes до успешного старта.
