# Kubernetes — Service и Ingress

## Проблема

Pod эфемерны — при пересоздании получают новый IP. Нельзя обращаться к Pod напрямую по IP. Нужен стабильный endpoint.

---

## Service

**Service** — абстракция, предоставляющая стабильный IP и DNS-имя для группы Pod. Выбирает Pod по label selector.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web          # все Pod с label app=web
  ports:
    - port: 80        # порт Service
      targetPort: 8000 # порт контейнера
  type: ClusterIP     # по умолчанию
```

При обращении к `web:80` трафик балансируется между Pod с label `app: web`.

---

## Типы Service

### ClusterIP (по умолчанию)

Доступен только внутри кластера. Получает виртуальный IP из диапазона Service CIDR.

```yaml
spec:
  type: ClusterIP
```

Используется для: внутренних сервисов (БД, кэш, межсервисное общение).

### NodePort

Открывает порт (30000–32767) на каждой ноде кластера:

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8000
      nodePort: 30080    # опционально, иначе случайный
```

```
Внешний трафик → <NodeIP>:30080 → Service → Pod:8000
```

Используется для: dev/test окружений, простых случаев без облака.

### LoadBalancer

Создаёт внешний балансировщик (через cloud provider):

```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8000
```

```
Интернет → Cloud LB (внешний IP) → NodePort → Service → Pod
```

Используется для: production в облаке. Каждый LoadBalancer Service = отдельный облачный LB (дорого, если много сервисов → используй Ingress).

### ExternalName

DNS-алиас для внешнего сервиса:

```yaml
spec:
  type: ExternalName
  externalName: db.example.com
```

`web` обращается к `db-service` → резолвится в `db.example.com`.

---

## Сводная таблица

| Тип | Доступность | Использование |
|---|---|---|
| ClusterIP | Только внутри кластера | Межсервисное общение |
| NodePort | Извне через `<NodeIP>:<port>` | Dev/test |
| LoadBalancer | Извне через облачный LB | Production (один LB на Service) |
| ExternalName | DNS-алиас | Интеграция с внешними сервисами |

---

## Headless Service

Service без виртуального IP (`clusterIP: None`). DNS возвращает IP всех Pod напрямую.

```yaml
spec:
  clusterIP: None
  selector:
    app: db
```

Используется с **StatefulSet** — клиент может обращаться к конкретному Pod (`db-0.db-headless.default.svc.cluster.local`).

---

## DNS в Kubernetes

Кластерный DNS (CoreDNS) создаёт записи для каждого Service:

```
<service>.<namespace>.svc.cluster.local
```

```bash
# Внутри Pod
curl http://web                          # тот же namespace
curl http://web.staging                  # другой namespace
curl http://web.staging.svc.cluster.local # полное имя (FQDN)
```

Для headless Service + StatefulSet — записи для каждого Pod:
```
<pod-name>.<service>.<namespace>.svc.cluster.local
# db-0.db-headless.default.svc.cluster.local
```

---

## Ingress

**Ingress** — правила маршрутизации HTTP/HTTPS трафика из внешнего мира к Service внутри кластера. Один Ingress заменяет множество LoadBalancer Service.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

**Ingress Controller** — реализация, которая читает Ingress-ресурсы и настраивает прокси:
- **nginx-ingress** — на базе nginx (самый популярный)
- **Traefik** — cloud-native, автоконфигурация
- **AWS ALB Ingress Controller** — для AWS
- **Istio Gateway** — в service mesh

```
Интернет → Ingress Controller (nginx) → Service → Pod
                   ↑
           читает Ingress-правила
```

---

## Ingress vs LoadBalancer Service

| | Ingress | LoadBalancer Service |
|---|---|---|
| Уровень | L7 (HTTP/HTTPS) | L4 (TCP/UDP) |
| Маршрутизация | По host/path | Нет |
| TLS termination | Да | Нет (нужен отдельный LB) |
| Стоимость | Один LB на кластер | LB на каждый Service |
| Применение | HTTP API, веб-приложения | TCP-сервисы (БД, gRPC) |

---

## Команды

```bash
kubectl get svc                          # список Service
kubectl get svc -o wide                  # с selector и endpoint
kubectl describe svc web                 # детали
kubectl get endpoints web                # IP Pod за Service
kubectl get ingress                      # список Ingress
kubectl describe ingress app-ingress     # детали
```

---

## Вопросы на собеседовании

**Зачем нужен Service, если можно обращаться к Pod по IP?**
Pod эфемерны — при пересоздании IP меняется. Service предоставляет стабильный IP и DNS-имя, балансирует трафик между Pod. Без Service нужно отслеживать IP каждого Pod вручную.

**Когда использовать Ingress вместо LoadBalancer Service?**
Когда есть несколько HTTP-сервисов. LoadBalancer создаёт отдельный облачный LB для каждого Service (дорого). Ingress — один LB + маршрутизация по host/path, TLS termination. Для не-HTTP трафика (TCP/gRPC) — LoadBalancer Service.

**Что такое headless Service и зачем он нужен?**
Service с `clusterIP: None`. DNS возвращает IP всех Pod напрямую, а не виртуальный IP. Нужен для StatefulSet (БД) — клиент может обращаться к конкретному Pod по имени (`db-0.db-svc`).

**Как Service находит Pod?**
Через label selector. Service выбирает Pod, чьи labels совпадают с selector. kube-proxy настраивает правила iptables/IPVS для маршрутизации на IP этих Pod. Endpoints обновляются автоматически при добавлении/удалении Pod.
