# Kubernetes — Основы

## Что такое Kubernetes

**Kubernetes (K8s)** — платформа оркестрации контейнеров. Автоматизирует развёртывание, масштабирование, управление и самовосстановление контейнеризованных приложений.

**Зачем нужен (проблемы без оркестратора):**
- Как распределить контейнеры по серверам?
- Что делать, если контейнер упал?
- Как масштабировать при росте нагрузки?
- Как обновлять без даунтайма?
- Как управлять сетью между сотнями контейнеров?

---

## Архитектура кластера

Кластер Kubernetes состоит из **Control Plane** (управляющий слой) и **Worker Nodes** (рабочие узлы).

```
┌─────────────────── Control Plane ───────────────────┐
│  kube-apiserver   etcd   kube-scheduler             │
│  kube-controller-manager   cloud-controller-manager │
└─────────────────────────────────────────────────────┘
        │                    │                  │
┌───────┴──────┐  ┌──────────┴─────┐  ┌────────┴───────┐
│  Worker Node │  │  Worker Node   │  │  Worker Node   │
│  kubelet     │  │  kubelet       │  │  kubelet       │
│  kube-proxy  │  │  kube-proxy    │  │  kube-proxy    │
│  container   │  │  container     │  │  container     │
│  runtime     │  │  runtime       │  │  runtime       │
│  [Pod] [Pod] │  │  [Pod] [Pod]   │  │  [Pod] [Pod]   │
└──────────────┘  └────────────────┘  └────────────────┘
```

### Control Plane

| Компонент | Назначение |
|---|---|
| **kube-apiserver** | Единая точка входа (REST API). Все компоненты общаются через него |
| **etcd** | Распределённое key-value хранилище. Хранит всё состояние кластера |
| **kube-scheduler** | Выбирает на какой Node запустить Pod (по ресурсам, affinity, taints) |
| **kube-controller-manager** | Запускает контроллеры (ReplicaSet, Deployment, Node, Job и др.) |
| **cloud-controller-manager** | Интеграция с облачными провайдерами (LoadBalancer, Volumes) |

### Worker Node

| Компонент | Назначение |
|---|---|
| **kubelet** | Агент на каждой ноде. Получает спецификации Pod от API server, запускает контейнеры |
| **kube-proxy** | Сетевой прокси. Управляет правилами маршрутизации (iptables/IPVS) для Service |
| **Container Runtime** | containerd / CRI-O — запускает контейнеры |

---

## Декларативная модель

Kubernetes работает по принципу **desired state** (желаемое состояние):
1. Ты описываешь желаемое состояние в YAML-манифесте
2. Отправляешь в API server (`kubectl apply`)
3. Контроллеры непрерывно приводят текущее состояние к желаемому (reconciliation loop)

```yaml
# "Хочу 3 реплики nginx"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  ...
```

Если Pod упал → контроллер создаст новый. Если нода недоступна → Pod перезапустится на другой.

---

## Pod

**Pod** — минимальная единица деплоя в Kubernetes. Один или несколько контейнеров с общими ресурсами.

**Контейнеры в Pod разделяют:**
- Сетевой namespace (один IP, видят друг друга через `localhost`)
- Volumes
- IPC namespace

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  labels:
    app: web
spec:
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 8000
      resources:
        requests:
          memory: "128Mi"
          cpu: "250m"
        limits:
          memory: "256Mi"
          cpu: "500m"
```

**Когда несколько контейнеров в Pod (sidecar pattern):**
- Logging agent (Fluentd) рядом с приложением
- Service mesh proxy (Envoy/Istio)
- Adapter — преобразование формата данных

**Pod — эфемерный:** не перезапускается, а пересоздаётся (с новым IP). Поэтому Pods не создают напрямую — используют контроллеры.

---

## ReplicaSet

**ReplicaSet** — гарантирует, что заданное количество идентичных Pod запущено в любой момент.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: app
          image: myapp:1.0
```

На практике ReplicaSet не создают напрямую — используют Deployment.

---

## Deployment

**Deployment** — управляет ReplicaSet и предоставляет декларативные обновления Pod.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # сколько Pod сверх replicas при обновлении
      maxUnavailable: 0     # сколько Pod может быть недоступно
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: app
          image: myapp:1.0
          ports:
            - containerPort: 8000
```

**Иерархия:** Deployment → ReplicaSet → Pod

```bash
kubectl apply -f deployment.yaml       # создать/обновить
kubectl get deployments                 # список
kubectl get pods                        # список Pod
kubectl describe deployment web         # детальная информация
kubectl rollout status deployment/web   # статус обновления
kubectl rollout history deployment/web  # история версий
kubectl rollout undo deployment/web     # откат к предыдущей версии
kubectl scale deployment web --replicas=5  # масштабирование
```

---

## Другие контроллеры

| Контроллер | Назначение |
|---|---|
| **StatefulSet** | Для stateful-приложений (БД). Стабильные имена (`web-0`, `web-1`), упорядоченный запуск, персистентные тома |
| **DaemonSet** | По одному Pod на каждой ноде (мониторинг, логирование) |
| **Job** | Одноразовая задача (миграции, batch). Завершается после выполнения |
| **CronJob** | Периодические задачи по расписанию (cron-синтаксис) |

---

## Namespaces

**Namespace** — логическое разделение кластера. Изоляция ресурсов, квот, RBAC.

```bash
kubectl get namespaces
kubectl create namespace staging
kubectl get pods -n staging              # Pod в конкретном namespace
kubectl get pods --all-namespaces        # во всех
```

**Встроенные namespaces:**
- `default` — по умолчанию
- `kube-system` — системные компоненты K8s
- `kube-public` — публичные ресурсы
- `kube-node-lease` — heartbeat нод

---

## Labels и Selectors

**Labels** — key-value метки на объектах. Используются для выборки и группировки.

```yaml
metadata:
  labels:
    app: web
    env: production
    version: v2
```

```bash
kubectl get pods -l app=web              # по label
kubectl get pods -l 'env in (prod,staging)'
```

**Selectors** — способ выбрать объекты по labels. Используются в Service, Deployment, ReplicaSet для связывания.

---

## kubectl — основные команды

```bash
# Информация
kubectl cluster-info
kubectl get nodes
kubectl get all                          # pods, services, deployments...
kubectl get pods -o wide                 # с IP и нодой
kubectl describe pod <name>              # детальное описание
kubectl logs <pod>                       # логи
kubectl logs <pod> -c <container>        # логи конкретного контейнера
kubectl logs -f <pod>                    # follow

# Управление
kubectl apply -f manifest.yaml           # создать/обновить (декларативно)
kubectl delete -f manifest.yaml          # удалить
kubectl exec -it <pod> -- bash           # shell внутри
kubectl port-forward <pod> 8080:80       # проброс порта на localhost

# Debug
kubectl get events --sort-by='.lastTimestamp'
kubectl top pods                         # потребление ресурсов
kubectl top nodes
```

---

## Вопросы на собеседовании

**Чем Kubernetes отличается от Docker?**
Docker — runtime для запуска контейнеров. Kubernetes — оркестратор, который управляет контейнерами в кластере: распределяет по нодам, масштабирует, перезапускает, обновляет, управляет сетью. K8s использует container runtime (containerd) для запуска контейнеров.

**Что такое Pod и почему это не просто контейнер?**
Pod — группа из одного или нескольких контейнеров с общей сетью (один IP), volumes и жизненным циклом. Позволяет запускать sidecar-контейнеры (логирование, proxy) рядом с основным приложением. Pod — эфемерный, не перезапускается, а пересоздаётся.

**Что произойдёт, если нода с Pod упадёт?**
kubelet перестанет отправлять heartbeat. Через ~5 минут (node controller) нода помечается NotReady. Pod с Deployment/ReplicaSet будут пересозданы scheduler-ом на другой ноде. Standalone Pod (без контроллера) — потерян.

**Зачем нужен ReplicaSet, если есть Deployment?**
Deployment — надстройка над ReplicaSet. При обновлении Deployment создаёт новый ReplicaSet и плавно переключает трафик (RollingUpdate). Старый ReplicaSet сохраняется для возможности отката. Напрямую ReplicaSet обычно не используют.

**Разница между Deployment и StatefulSet?**
Deployment — для stateless-приложений (все Pod взаимозаменяемы). StatefulSet — для stateful (БД): стабильные сетевые имена (`app-0`, `app-1`), упорядоченный запуск/остановка, каждый Pod привязан к своему PersistentVolume.

**Как работает декларативная модель Kubernetes?**
Ты описываешь желаемое состояние (desired state) в YAML. Контроллеры в цикле сравнивают текущее состояние с желаемым и выполняют действия для приведения к нему (reconciliation loop). Если Pod упал — контроллер создаст новый.
