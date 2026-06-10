---
name: hyranse-postgresql-deploy
description: Deploy PostgreSQL via Helm chart for stage/prod + auto weekly backup to Backblaze B2 for prod.
disable-model-invocation: true
---

# Hyranse PostgreSQL — CI/CD Deployment

Источник истины: проекты с `deploy/postgresql-chart/` и `deploy/postgresql-application*.yaml`.

## Архитектура

```
<project-root>/
├── deploy/
│   ├── postgresql-chart/              # Helm chart (обёртка над bitnami/postgresql)
│   │   ├── Chart.yaml
│   │   ├── values.yaml                # общие значения (образ, auth, storage, probes)
│   │   ├── values-stage.yaml          # stage overrides (NodePort)
│   │   └── values-prod.yaml           # prod overrides (NodePort + backup)
│   ├── postgresql-application.yaml    # ArgoCD Application — prod
│   ├── postgresql-application-stage.yaml  # ArgoCD Application — stage
│   ├── postgresql-backup-cronjob.yaml # CronJob — только для prod
│   └── postgresql-backup-app.yaml     # ArgoCD Application — backup
```

## Соглашение об именовании

Все ArgoCD Application и Kubernetes-ресурсы должны следовать единой схеме (см. [hyranse-backend-deploy SKILL.md](../hyranse-backend-deploy/SKILL.md) для полного описания).

### Шаблон

```
<project>-<component>[-<env>][-<aux>]
```

| Часть | Описание | Примеры |
|-------|----------|---------|
| `project` | Название проекта | `hyranse-email`, `hyranse-backend` |
| `component` | Узел приложения | `postgresql` |
| `env` (опционально) | Окружение: пусто = prod, `stage` | пусто, `stage` |
| `aux` (опционально) | Доп. назначение | `backup`, `backup-prod` |

### Примеры для PostgreSQL

| ArgoCD Application | Назначение |
|--------------------|-----------|
| `hyranse-email-postgresql` | PostgreSQL prod |
| `hyranse-email-postgresql-stage` | PostgreSQL stage |
| `hyranse-email-postgresql-backup` | Backup CronJob prod (в `k8s_argo`) |
| `hyranse-email-postgresql-backup-prod` | Backup CronJob prod (в корне проекта) |

### Правила

- **prod** — без суффикса `env`: `hyranse-email-postgresql`, не `hyranse-email-postgresql-prod`
- **stage** — суффикс `-stage`: `hyranse-email-postgresql-stage`
- **backup prod** — суффикс `-backup` (или `-backup-prod` если есть backup-stage)
- **namespace prod** — `hyranse-<project>` (напр. `hyranse-email`)
- **namespace stage** — `hyranse-stage`

## Принципы

1. **Helm chart** — обёртка над `bitnami/postgresql` (`oci://registry-1.docker.io/bitnamicharts`). Все настройки через values.
2. **Два окружения**: stage и prod — каждый в своём namespace.
   - stage: `hyranse-stage`, NodePort
   - prod: `hyranse-<project>`, NodePort + TLS (при необходимости)
3. **Backup (только prod)** — CronJob раз в неделю: `pg_dump` → gzip → Backblaze B2 bucket.
4. **Без управления бэкапами через ArgoCD** — backup-приложение может быть в `k8s_argo` или в корне проекта (по выбору).

## Этапы настройки

### 1. Создать Helm chart `deploy/postgresql-chart/`

Chart.yaml — зависимость от bitnami/postgresql:

```yaml
apiVersion: v2
name: <project-name>-postgresql
description: PostgreSQL for <project description>
type: application
version: 0.1.0
appVersion: "16.3.0"
dependencies:
  - name: postgresql
    version: 16.2.1
    repository: oci://registry-1.docker.io/bitnamicharts
```

### 2. Создать values-файлы

**`values.yaml` (базовые)**:

```yaml
postgresql:
  fullnameOverride: <project-name>-postgresql

  auth:
    username: postgresql
    password: postgresql
    database: postgresql
    postgresPassword: postgresql
    enablePostgresUser: true

  global:
    imageRegistry: docker.io
    image:
      registry: docker.io
      repository: bitnamilegacy/postgresql
      tag: 16.3.0-debian-12-r10

  image:
    registry: docker.io
    repository: bitnamilegacy/postgresql
    tag: 16.3.0-debian-12-r10

  primary:
    image:
      registry: docker.io
      repository: bitnamilegacy/postgresql
      tag: 16.3.0-debian-12-r10
    service:
      type: NodePort
      nodePorts:
        postgresql: 30000   # placeholder, переопределяется в stage/prod
    persistence:
      enabled: true
      size: 8Gi
      storageClass: ""
      accessModes:
        - ReadWriteOnce
    resources:
      requests:
        cpu: 100m
        memory: 384Mi
      limits:
        cpu: 1000m
        memory: 1Gi
    extendedConfiguration: |-
      max_connections = 200
    initdb:
      scripts:
        init-<project>.sh: |
          #!/bin/bash
          set -e
          psql -v ON_ERROR_STOP=1 --username "postgres" <<-EOSQL
            SELECT 'CREATE DATABASE <db_name>'
            WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = '<db_name>')\gexec
            GRANT ALL PRIVILEGES ON DATABASE <db_name> TO postgres;
            GRANT ALL PRIVILEGES ON DATABASE <db_name> TO postgresql;
          EOSQL
          psql -v ON_ERROR_STOP=1 --username "postgres" --dbname "<db_name>" <<-EOSQL
            GRANT ALL ON SCHEMA public TO postgresql;
            ALTER SCHEMA public OWNER TO postgresql;
            GRANT CREATE ON SCHEMA public TO postgresql;
          EOSQL
```

**`values-stage.yaml`**:
```yaml
postgresql:
  primary:
    service:
      nodePorts:
        postgresql: 3011X   # уникальный порт для stage
```

**`values-prod.yaml`**:
```yaml
postgresql:
  primary:
    service:
      nodePorts:
        postgresql: 3010X   # уникальный порт для prod
```

### 3. Создать ArgoCD Application-манифесты

**`deploy/postgresql-application.yaml`** (prod):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <project-name>-postgresql
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    repoURL: https://github.com/<repo-owner>/<repo-name>.git
    targetRevision: HEAD
    path: deploy/postgresql-chart
    helm:
      releaseName: <project-name>-postgresql
      valueFiles:
        - values.yaml
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: hyranse-<project>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**`deploy/postgresql-application-stage.yaml`** (stage):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <project-name>-postgresql-stage
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    repoURL: https://github.com/<repo-owner>/<repo-name>.git
    targetRevision: stage  # или HEAD, в зависимости от стратегии веток
    path: deploy/postgresql-chart
    helm:
      releaseName: <project-name>-postgresql
      valueFiles:
        - values.yaml
        - values-stage.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: hyranse-stage
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 4. Настроить автоматический бэкап (только prod)

Создать **`deploy/postgresql-backup-cronjob.yaml`**:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: <project-name>-postgresql-backup
  namespace: hyranse-<project>
spec:
  # Раз в неделю — воскресенье в 02:00
  schedule: "0 2 * * 0"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: <project-name>-postgresql-backup
          containers:
          - name: postgres-backup
            image: postgres:16
            command:
            - /bin/bash
            - -c
            - |
              set -e
              echo "Installing B2 CLI..."
              apt-get update -qq && apt-get install -y -qq python3 python3-pip curl > /dev/null 2>&1
              pip3 install --quiet --upgrade b2 --break-system-packages

              echo "Starting PostgreSQL backup..."

              # Authenticate with Backblaze
              b2 account authorize $B2_APPLICATION_KEY_ID $B2_APPLICATION_KEY

              # Create timestamp for backup filename
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)

              # Имя файла = имя ресурса PostgreSQL + дата
              # Например: hyranse-backend-postgresql_20260610_020000.sql.gz
              BACKUP_FILE="<project-name>-postgresql_${TIMESTAMP}.sql"

              # Create PostgreSQL dump
              echo "Creating database dump..."
              PGPASSWORD=$DB_PASSWORD pg_dump -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME > /tmp/$BACKUP_FILE

              # Compress the backup
              echo "Compressing backup..."
              gzip /tmp/$BACKUP_FILE
              BACKUP_FILE="${BACKUP_FILE}.gz"

              # Upload to Backblaze
              echo "Uploading to Backblaze..."
              b2 file upload $B2_BUCKET_ID /tmp/$BACKUP_FILE $BACKUP_FILE

              echo "Backup completed successfully: $BACKUP_FILE"

              # Cleanup old backups (keep last 30 days)
              echo "Cleaning up old backups..."
              RETENTION_DAYS=30
              CUTOFF_DATE=$(date -d "$RETENTION_DAYS days ago" +%Y%m%d)

              b2 ls b2://$B2_BUCKET_ID | grep "<project-name>-postgresql_" | while read -r line; do
                FILENAME=$(echo "$line" | awk '{print $1}')
                FILE_DATE=$(echo "$FILENAME" | grep -oP '\d{8}' | head -1)
                if [ ! -z "$FILE_DATE" ] && [ "$FILE_DATE" -lt "$CUTOFF_DATE" ]; then
                  echo "Deleting old backup: $FILENAME"
                  b2 file delete $B2_BUCKET_ID $FILENAME || true
                fi
              done

              echo "Cleanup completed"
              rm -f /tmp/$BACKUP_FILE
            env:
            - name: DB_HOST
              value: "<project-name>-postgresql"
            - name: DB_PORT
              value: "5432"
            - name: DB_USER
              value: "postgresql"
            - name: DB_NAME
              value: "postgresql"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: <project-name>-postgresql
                  key: password
            - name: B2_APPLICATION_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: backblaze-credentials
                  key: keyID
            - name: B2_APPLICATION_KEY
              valueFrom:
                secretKeyRef:
                  name: backblaze-credentials
                  key: applicationKey
            - name: B2_BUCKET_ID
              valueFrom:
                secretKeyRef:
                  name: backblaze-credentials
                  key: bucketID
```

> **Важно**: Secret `backblaze-credentials` должен существовать в namespace до применения CronJob. Если его нет — создать вручную или через ArgoCD Application из `k8s_argo/hyranse/postgresql-backup-cronjob.yaml`.

Также нужен **ServiceAccount** с корректным именем (или убрать `serviceAccountName`, если не требуется):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <project-name>-postgresql-backup
  namespace: hyranse-<project>
---
# Если namespace использует PodSecurity или ограничения — добавить RoleBinding
```

### 5. Создать ArgoCD Application для бэкапа (опционально)

Бэкап можно добавить в ArgoCD двумя способами:

**A. В корне проекта** — `deploy/postgresql-backup-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <project-name>-postgresql-backup
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  source:
    repoURL: https://github.com/<repo-owner>/<repo-name>.git
    targetRevision: HEAD
    path: deploy
    directory:
      include: postgresql-backup-cronjob.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: hyranse-<project>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**B. В `k8s_argo`** — добавить в `hyranse/` директорию и создать Application с `include: postgresql-backup-cronjob.yaml`.

## Что делает бэкап

| Параметр | Значение |
|----------|----------|
| Расписание | Каждое воскресенье в 02:00 (`0 2 * * 0`) |
| Инструмент | `pg_dump` (полный дамп) |
| Сжатие | gzip |
| Хранилище | Backblaze B2 (bucket: `hyranse-postges`) |
| Имя файла | `<resource-name>_<YYYYMMDD_HHMMSS>.sql.gz` |
| Хранение | 30 дней (старые удаляются) |
| Пароль БД | Берётся из Secret, созданного Helm chart'ом (`<fullnameOverride>` → secret name) |

## Примеры имён файлов

- `hyranse-backend-postgresql_20260610_020000.sql.gz`
- `hyranse-email-postgresql_20260603_020000.sql.gz`

## Чего НЕ делать в этом скилле

- Не создавать несколько PostgreSQL в одном namespace (если не нужно)
- Не настраивать репликацию / streaming (single primary)
- Не менять пароли в values — использовать дефолтные (они в Secret'е после деплоя)
- Не создавать сам bucket в Backblaze — он уже существует
- Не настраивать cert-manager или ingress (PostgreSQL не требует Ingress)
- Stage бэкапы не делать (если явно не запрошено)

## Чеклист

- [ ] Создан `deploy/postgresql-chart/Chart.yaml` с зависимостью от bitnami/postgresql
- [ ] Создан `deploy/postgresql-chart/values.yaml` (образ, auth, storage, resources, initdb)
- [ ] Создан `deploy/postgresql-chart/values-stage.yaml` (уникальный NodePort)
- [ ] Создан `deploy/postgresql-chart/values-prod.yaml` (уникальный NodePort)
- [ ] Выполнен `helm dependency update deploy/postgresql-chart/`
- [ ] Создан `deploy/postgresql-application.yaml` (prod, sync-wave: "1")
- [ ] Создан `deploy/postgresql-application-stage.yaml` (stage, sync-wave: "1")
- [ ] Создан `deploy/postgresql-backup-cronjob.yaml` (только prod, weekly)
- [ ] Проверено, что Secret `backblaze-credentials` существует в prod namespace
- [ ] Убедиться, что NodePort не конфликтует с другими сервисами (список занятых портов см. ниже)
- [ ] Закоммитить и запушить → проверить ArgoCD (Sync + Healthy)

### Занятые NodePort для PostgreSQL

| Проект | Stage | Prod |
|--------|-------|------|
| hyranse-backend | 30115 | 30105 |
| hyranse-email | 30116 | 30103 |

При добавлении нового проекта — выбрать следующий свободный порт в диапазоне 30100-30199.
