---
name: hyranse-backend-deploy
description: Configure CI/CD for hyranse_backend_main: GitHub Actions, Helm charts, stage/prod branches, and ArgoCD sync.
disable-model-invocation: true
---

# Hyranse Backend — CI/CD Setup

Источник истины: `hyranse_backend_main/.github/workflows/` и `hyranse_backend_main/deploy/`.

## Архитектура деплоя

```
hyranse_backend_main/
├── .github/workflows/
│   ├── ci-docker-publish.yml    # сборка по git-тегам (v*.*.*)
│   ├── docker-ghcr.yml          # prod: тесты → сборка → деплой (ветка prod)
│   └── docker-stage.yml         # stage: тесты → сборка → деплой (ветка stage)
├── deploy/
│   ├── application.yaml         # ArgoCD Application — prod
│   ├── application-stage.yaml   # ArgoCD Application — stage
│   ├── postgresql-application.yaml
│   ├── postgresql-application-stage.yaml
│   ├── postgresql-chart/
│   ├── postgresql-backup-app.yaml
│   ├── postgresql-backup-cronjob.yaml
│   ├── opensearch-credentials-prod-secret.yaml
│   ├── opensearch-credentials-prod-secret-backend.yaml
│   └── chart/                   # Helm chart приложения
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-stage.yaml
│       ├── values-prod.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ingress.yaml
└── Dockerfile
```

## Flow деплоя

```
push в prod/stage
  → GitHub Actions (тесты, docker build, push GHCR)
  → sed image.tag в values-prod.yaml / values-stage.yaml
  → commit + push в ту же ветку
  → curl ArgoCD sync API
  → ArgoCD подтягивает chart из ветки prod/stage
```

| Окружение | Git-ветка | Workflow | ArgoCD app | values file | Image tags |
|-----------|-----------|----------|------------|-------------|------------|
| prod | `prod` | `docker-ghcr.yml` | `hyranse-backend` | `values-prod.yaml` | `latest`, `<sha>` |
| stage | `stage` | `docker-stage.yml` | `hyranse-backend-stage` | `values-stage.yaml` | `stage-latest`, `stage-<sha>` |

Tag в Helm values: prod — `<sha>`, stage — `stage-<sha>`.

## Принципы

1. **GitHub Actions** — CI/CD (тесты, сборка образа, пуш в GHCR, обновление Helm values, триггер ArgoCD sync).
2. **Helm chart** — шаблоны Kubernetes (Deployment, Service, Ingress). Env-переменные через values.
3. **Две ветки**: `stage` и `prod`. Каждая ветка — отдельный pipeline и ArgoCD Application.
4. **GHCR** (ghcr.io) — registry для Docker-образов.
5. **ArgoCD** — GitOps, `targetRevision` = имя ветки (`prod` / `stage`), не HEAD.

## Соглашение об именовании

```
<project>-<component>[-<env>][-<aux>]
```

| Часть | Описание | Примеры |
|-------|----------|---------|
| `project` | Название проекта | `hyranse-email`, `hyranse-backend` |
| `component` | Узел приложения | `backend`, `frontend`, `postgresql` |
| `env` (опционально) | Окружение: пусто = prod, `stage` | пусто, `stage` |
| `aux` (опционально) | Доп. назначение | `backup`, `backup-prod` |

| Ресурс | Назначение |
|--------|-----------|
| `hyranse-backend` | Backend prod |
| `hyranse-backend-stage` | Backend stage |
| `hyranse-backend-postgresql` | PostgreSQL prod |
| `hyranse-backend-postgresql-stage` | PostgreSQL stage |

### Правила

- **prod** — без суффикса `env`
- **stage** — суффикс `-stage` (всегда в конце)
- **namespace prod** — `hyranse-<project>` (напр. `hyranse-backend`)
- **namespace stage** — `hyranse-stage` (единый для всех stage-сервисов)
- **Git-ветка** `prod` → prod, `stage` → stage

## Этапы настройки CI/CD

### 1. Создать GitHub Actions workflows

Три файла:

- **`ci-docker-publish.yml`** — сборка образа при git-теге `v*.*.*`. Теги: `latest`, `v<version>`, `sha-<commit>`.
- **`docker-ghcr.yml`** — prod pipeline при пуше в `prod`.
- **`docker-stage.yml`** — stage pipeline при пуше в `stage`.

Шаблоны: [reference-github-actions.md](reference-github-actions.md).

### 2. Создать Helm chart

Шаблоны: [reference-helm-chart.md](reference-helm-chart.md).

Ключевые особенности:
- `opensearchUrlSecret` — URL OpenSearch из K8s Secret (нельзя одновременно literal в env и secret)
- `linkedinParserDbPasswordSecret` — пароль parser DB из Secret (опционально)
- Ingress в prod, NodePort в stage (по умолчанию)

### 3. Создать ветки stage и prod

- **`stage`** — ArgoCD `hyranse-backend-stage`, `values-stage.yaml`, `targetRevision: stage`
- **`prod`** — ArgoCD `hyranse-backend`, `values-prod.yaml`, `targetRevision: prod`

### 4. Создать ArgoCD Application-манифесты

```
deploy/application.yaml        # prod, targetRevision: prod
deploy/application-stage.yaml  # stage, targetRevision: stage
```

Шаблоны: [reference-argo.md](reference-argo.md).

### 5. Связанная инфраструктура (не генерировать в скилле, но должна существовать)

Для `hyranse_backend_main` в `deploy/` уже есть отдельные Application:
- PostgreSQL (`postgresql-application*.yaml`, `postgresql-chart/`)
- Backup CronJob (`postgresql-backup-*.yaml`)
- OpenSearch credentials (`opensearch-credentials-*.yaml`)

Скилл не создаёт эти ресурсы, но приложение от них зависит.

## Настройка Ingress (опционально)

Разделять **Ingress host в chart** и **публичный backend URL** (`env.backendUrl`).

| Окружение | Ingress в chart | Доступ | Namespace |
|-----------|-----------------|--------|-----------|
| prod | `backend.hyranse.com` (enabled) | Ingress + TLS | `hyranse-backend` |
| stage | disabled (наследует `values.yaml`) | NodePort `30091` | `hyranse-stage` |

Публичные URL приложения (через внешний API gateway):
- prod: `https://api.hyranse.com/backend-main`
- stage: `https://stage-api.hyranse.com/backend-main`

При активации скилла агент запрашивает:

- **Включить Ingress для prod?** (Y/n)
- **Домен prod** (host): по умолчанию `backend.hyranse.com`
- **Включить Ingress для stage?** (y/N) — по умолчанию нет, NodePort
- **Включить TLS через cert-manager?** (Y/n) — только для prod Ingress
- **Имя TLS-секрета**

## Health checks и Probes

### Эндпоинты

| Probe | Path | Назначение |
|-------|------|------------|
| startup | `/ready` | Полный старт (БД, зависимости) |
| liveness | `/healthz` | Процесс жив, без внешних зависимостей |
| readiness | `/ready` | Готов принимать трафик (PostgreSQL, OpenSearch, Redis и др.) |

Референс: `Hyranse_email_plugin/backend/src/health/`.

Модуль в приложении:

```
src/health/
├── health.module.ts
├── health.controller.ts
└── health.service.ts
```

### Значения по умолчанию

| Probe | Path | initialDelaySeconds | periodSeconds | timeoutSeconds | failureThreshold |
|-------|------|---------------------|---------------|----------------|-----------------|
| startup | `/ready` | 5 | 5 | 5 | 36 |
| liveness | `/healthz` | 15 | 60 | 5 | 3 |
| readiness | `/ready` | 5 | 10 | 5 | 3 |

## Prometheus monitoring

Источник истины: `k8s_argo/observability/prometeus/`. Перед добавлением — проверить, нет ли уже job/alert для сервиса.

Шаблоны: [reference-prometheus.md](reference-prometheus.md).

## Чего НЕ делать в этом скилле

- Не создавать PostgreSQL, OpenSearch, Redis (но для backend_main манифесты уже в `deploy/`)
- Не настраивать cert-manager, ingress-nginx, сам ArgoCD
- Не создавать секреты в Kubernetes
- Не коммитить реальные токены, пароли, API keys
- Не писать бизнес-логику (health-модуль `src/health/` — исключение)

## Чеклист

- [ ] Dockerfile в корне проекта
- [ ] Health-модуль (`src/health/`) с `/healthz` и `/ready`
- [ ] Workflows: `ci-docker-publish.yml`, `docker-ghcr.yml`, `docker-stage.yml`
- [ ] Helm chart с probes, `opensearchUrlSecret`, env-переменными
- [ ] `values-stage.yaml` и `values-prod.yaml`
- [ ] ArgoCD Applications с `targetRevision: stage` / `prod`
- [ ] Ветки `stage` и `prod` в репозитории
- [ ] GitHub Secrets: `ARGOCD_TOKEN`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` (опционально)
- [ ] GitHub Variables: `ARGOCD_BASE_URL`
- [ ] `imagePullSecrets` (ghcr-cred) в namespace'ах
- [ ] Prometheus blackbox job и alerting rules (если нужен мониторинг)

## Параметры ArgoCD (GitHub)

| Параметр | Где задать |
|----------|------------|
| `ARGOCD_BASE_URL` | GitHub Repository Variables |
| `ARGOCD_TOKEN` | GitHub Repository Secrets |

Не коммитить значения в репозиторий.

## Референсные значения для hyranse_backend_main

| Параметр | prod | stage |
|----------|------|-------|
| ArgoCD app | `hyranse-backend` | `hyranse-backend-stage` |
| Namespace | `hyranse-backend` | `hyranse-stage` |
| NodePort | `30081` | `30091` |
| Ingress host | `backend.hyranse.com` | disabled |
| GHCR image | `ghcr.io/ezenkof/hyranse_backend_main` | тот же |

## Полные шаблоны

- GitHub Actions: [reference-github-actions.md](reference-github-actions.md)
- Helm Chart: [reference-helm-chart.md](reference-helm-chart.md)
- ArgoCD Application: [reference-argo.md](reference-argo.md)
- Prometheus + Blackbox: [reference-prometheus.md](reference-prometheus.md)
