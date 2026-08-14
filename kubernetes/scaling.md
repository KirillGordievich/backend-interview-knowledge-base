# Kubernetes — Масштабирование и обновления

## Стратегии обновления (Deployment)

### RollingUpdate (по умолчанию)

Постепенная замена Pod: создаются новые → удаляются старые. Без даунтайма.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # +1 Pod сверх replicas при обновлении
      maxUnavailable: 0      # все Pod доступны (zero downtime)
```

```
v1: [Pod1] [Pod2] [Pod3]
     ↓ создаёт Pod4 (v2)
v1: [Pod1] [Pod2] [Pod3]  v2: [Pod4]
     ↓ удаляет Pod1
v1: [Pod2] [Pod3]  v2: [Pod4]
     ↓ ...
v2: [Pod4] [Pod5] [Pod6]
```

### Recreate

Все старые Pod удаляются, затем создаются новые. Есть даунтайм.

```yaml
spec:
  strategy:
    type: Recreate
```

Применение: приложение не поддерживает параллельную работу двух версий (миграции БД, эксклюзивная блокировка).

---

## Rollout-команды

```bash
# Обновление (меняем образ)
kubectl set image deployment/web app=myapp:2.0

# Или через apply
kubectl apply -f deployment.yaml

# Статус обновления
kubectl rollout status deployment/web

# История ревизий
kubectl rollout history deployment/web
kubectl rollout history deployment/web --revision=2

# Откат
kubectl rollout undo deployment/web                # к предыдущей ревизии
kubectl rollout undo deployment/web --to-revision=1 # к конкретной

# Пауза/продолжение (canary вручную)
kubectl rollout pause deployment/web
kubectl rollout resume deployment/web
```

---

## Blue-Green и Canary

Kubernetes не имеет встроенной поддержки, но можно реализовать:

### Blue-Green

Два Deployment (blue и green). Service переключается между ними через selector.

```yaml
# Blue (текущий)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-blue
spec:
  selector:
    matchLabels:
      app: web
      version: blue
  template:
    metadata:
      labels:
        app: web
        version: blue
    ...

# Service указывает на blue
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
    version: blue       # переключить на green для деплоя
```

### Canary

Два Deployment с одинаковыми labels, разное количество реплик:

```yaml
# Stable: 9 реплик
kind: Deployment
metadata:
  name: web-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web

# Canary: 1 реплика (новая версия)
kind: Deployment
metadata:
  name: web-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
```

Service с selector `app: web` → ~10% трафика идёт на canary. Для точного контроля — Istio, Argo Rollouts.

---

## Horizontal Pod Autoscaler (HPA)

Автоматически масштабирует количество Pod на основе метрик.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70     # масштабировать при >70% CPU
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```bash
# Или через команду
kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=70

kubectl get hpa
kubectl describe hpa web-hpa
```

**Требования:** Metrics Server должен быть установлен в кластере. Pod должны иметь resource requests.

**Алгоритм:**
```
desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))
```

HPA проверяет метрики каждые 15 секунд (по умолчанию). Cooldown: scale-up — 3 мин, scale-down — 5 мин.

---

## Vertical Pod Autoscaler (VPA)

Автоматически настраивает requests/limits контейнеров на основе реального потребления.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: Auto          # Auto, Off, Initial
```

**updateMode:**
- `Off` — только рекомендации (безопасно для начала)
- `Initial` — устанавливает при создании Pod
- `Auto` — пересоздаёт Pod с новыми requests

**HPA vs VPA:**

| | HPA | VPA |
|---|---|---|
| Масштабирует | Количество Pod | Ресурсы Pod (requests/limits) |
| Когда | Высокая нагрузка → больше Pod | Неправильные requests → корректировка |
| Совместимость | Не использовать вместе по CPU | Можно комбинировать (HPA по custom metrics + VPA) |

---

## Taints и Tolerations

**Taints** — метки на ноде, отталкивающие Pod. **Tolerations** — разрешение Pod запускаться на tainted ноде.

```bash
# Пометить ноду
kubectl taint nodes node1 gpu=true:NoSchedule
```

```yaml
# Pod, который может запускаться на этой ноде
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

**Effects:**
- `NoSchedule` — новые Pod без toleration не размещаются
- `PreferNoSchedule` — мягкое ограничение
- `NoExecute` — существующие Pod без toleration выселяются

---

## Node Affinity

Правила размещения Pod на нодах по labels:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # жёсткое
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
      preferredDuringSchedulingIgnoredDuringExecution: # мягкое
        - weight: 1
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values: ["eu-west-1a"]
```

---

## Pod Disruption Budget (PDB)

Гарантирует минимальное количество доступных Pod при добровольных disruptions (drain ноды, обновления кластера).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2              # или maxUnavailable: 1
  selector:
    matchLabels:
      app: web
```

---

## Команды масштабирования

```bash
kubectl scale deployment web --replicas=5
kubectl get hpa
kubectl top pods                          # потребление ресурсов (нужен Metrics Server)
kubectl top nodes
kubectl describe node <name>              # capacity, allocatable, pods
kubectl drain node1 --ignore-daemonsets    # эвакуировать Pod с ноды
kubectl cordon node1                       # запретить размещение новых Pod
kubectl uncordon node1                     # разрешить
```

---

## Вопросы на собеседовании

**Чем RollingUpdate отличается от Recreate?**
RollingUpdate постепенно заменяет Pod — создаёт новые, удаляет старые, без даунтайма. Recreate сначала удаляет все старые Pod, потом создаёт новые — есть даунтайм. Recreate используют, когда две версии не могут работать параллельно.

**Как работает HPA?**
HPA периодически (каждые 15 секунд) запрашивает метрики у Metrics Server. Сравнивает текущее потребление ресурсов с целевым. Рассчитывает нужное количество реплик и масштабирует Deployment. Требует resource requests на Pod.

**Как сделать canary-деплой в Kubernetes?**
Базовый: два Deployment с одинаковым label selector, разное количество реплик (9 stable + 1 canary ≈ 10% трафика). Продвинутый: Istio VirtualService (точное распределение по весу), Argo Rollouts (automated canary с анализом метрик).

**Что такое PodDisruptionBudget?**
PDB гарантирует минимальное количество доступных Pod при добровольных disruptions — drain ноды, обновление кластера. Например, `minAvailable: 2` не даст evict Pod, если останется менее 2. Не защищает от involuntary disruptions (крэш ноды).

**Зачем нужны taints и tolerations?**
Taints на ноде отталкивают Pod — запрещают размещение. Tolerations в Pod разрешают запуск на tainted ноде. Применение: выделенные ноды для GPU-задач, изоляция нод для конкретных workloads, master-ноды (taint по умолчанию).

**Как откатить обновление?**
`kubectl rollout undo deployment/web` — к предыдущей ревизии. `kubectl rollout undo deployment/web --to-revision=N` — к конкретной. Kubernetes хранит историю ReplicaSet-ов (по умолчанию 10 ревизий, `revisionHistoryLimit`).
