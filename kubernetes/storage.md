# Kubernetes — Хранение данных

## Проблема

Файловая система контейнера эфемерна — данные теряются при перезапуске Pod. Для персистентного хранения Kubernetes предоставляет систему Volumes.

---

## Volumes (уровень Pod)

Volume определяется в спецификации Pod и доступен контейнерам внутри него.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: data
          mountPath: /app/data
        - name: config
          mountPath: /app/config
          readOnly: true
  volumes:
    - name: data
      emptyDir: {}
    - name: config
      configMap:
        name: app-config
```

### Типы volumes

| Тип | Описание | Жизненный цикл |
|---|---|---|
| `emptyDir` | Пустая директория, общая для контейнеров в Pod | Удаляется с Pod |
| `hostPath` | Путь на ноде (для dev/test, не для production) | Привязан к ноде |
| `configMap` | Данные из ConfigMap как файлы | — |
| `secret` | Данные из Secret как файлы | — |
| `persistentVolumeClaim` | Ссылка на PVC (персистентное хранилище) | Независимый |

### emptyDir

Создаётся при запуске Pod, удаляется при удалении Pod. Общий для всех контейнеров в Pod.

```yaml
volumes:
  - name: shared-data
    emptyDir: {}

  - name: cache
    emptyDir:
      medium: Memory     # tmpfs — хранение в RAM
      sizeLimit: 100Mi
```

Применение: обмен данными между контейнерами в Pod (sidecar), временный кэш.

---

## PersistentVolume (PV) и PersistentVolumeClaim (PVC)

Абстракция для отделения хранилища от Pod:

```
Администратор создаёт PV → Разработчик запрашивает PVC → Pod использует PVC
```

### PersistentVolume (PV)

Кусок хранилища в кластере, созданный администратором или динамически через StorageClass.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:                        # для dev, в production — облачный диск
    path: /data/pv-data
```

### PersistentVolumeClaim (PVC)

Запрос на хранилище от разработчика. Kubernetes находит подходящий PV и привязывает.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

### Использование PVC в Pod

```yaml
spec:
  containers:
    - name: db
      image: postgres:16
      volumeMounts:
        - name: db-storage
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: db-storage
      persistentVolumeClaim:
        claimName: db-data
```

---

## Access Modes

| Режим | Сокращение | Описание |
|---|---|---|
| ReadWriteOnce | RWO | Чтение/запись одной нодой |
| ReadOnlyMany | ROX | Только чтение, несколько нод |
| ReadWriteMany | RWX | Чтение/запись, несколько нод |

Не все типы хранилищ поддерживают все режимы. Облачные блочные диски (EBS, Persistent Disk) обычно поддерживают только RWO. Для RWX нужны NFS, EFS, CephFS.

---

## Reclaim Policy

Что происходит с PV после удаления PVC:

| Политика | Поведение |
|---|---|
| **Retain** | PV сохраняется, данные остаются. Требует ручной очистки |
| **Delete** | PV и данные удаляются (по умолчанию для динамических PV) |
| **Recycle** | Deprecated. Очистка данных (`rm -rf /volume/*`) |

---

## StorageClass (динамическое выделение)

**StorageClass** позволяет автоматически создавать PV при запросе PVC (dynamic provisioning). Не нужно создавать PV вручную.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```yaml
# PVC — PV создастся автоматически
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  storageClassName: fast         # ссылка на StorageClass
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

**volumeBindingMode:**
- `Immediate` — PV создаётся сразу при создании PVC
- `WaitForFirstConsumer` — PV создаётся когда Pod зашедулен на ноду (учитывает зону доступности)

---

## StatefulSet + PVC

StatefulSet автоматически создаёт PVC для каждого Pod через `volumeClaimTemplates`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 3
  serviceName: postgres
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast
        resources:
          requests:
            storage: 10Gi
```

Создаст PVC: `data-postgres-0`, `data-postgres-1`, `data-postgres-2`. Каждый Pod привязан к своему тому.

---

## Команды

```bash
kubectl get pv                           # PersistentVolumes
kubectl get pvc                          # PersistentVolumeClaims
kubectl get sc                           # StorageClasses
kubectl describe pv pv-data
kubectl describe pvc db-data
kubectl delete pvc db-data               # удаление PVC (PV зависит от reclaim policy)
```

---

## Вопросы на собеседовании

**Чем PV отличается от PVC?**
PV — физический ресурс хранилища в кластере (создаётся администратором или динамически). PVC — запрос на хранилище от разработчика. PVC привязывается к подходящему PV по размеру, access mode и StorageClass. Разделение позволяет разработчику не знать деталей реализации хранилища.

**Что такое StorageClass и зачем он нужен?**
StorageClass описывает тип хранилища (SSD, HDD, NFS) и позволяет автоматически создавать PV при запросе PVC (dynamic provisioning). Без StorageClass администратор должен вручную создавать PV для каждого запроса.

**Что произойдёт с данными при удалении Pod?**
Зависит от типа volume. emptyDir — удалится вместе с Pod. PVC — данные сохранятся, новый Pod может подключить тот же PVC. При удалении PVC — зависит от reclaim policy PV (Retain сохраняет данные, Delete удаляет).

**Почему StatefulSet использует volumeClaimTemplates?**
Каждый Pod в StatefulSet нуждается в своём персистентном томе (например, каждая реплика PostgreSQL хранит свои данные). `volumeClaimTemplates` автоматически создаёт отдельный PVC для каждого Pod (`data-db-0`, `data-db-1`).
