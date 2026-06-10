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
│   └── docker-ghcr.yml          # сборка + деплой по пушам в main
├── deploy/
│   ├── application.yaml         # ArgoCD Application — prod
│   ├── application-stage.yaml   # ArgoCD Application — stage
│   └── chart/                   # Helm chart
│       ├── Chart.yaml           # имя chart, версия
│       ├── values.yaml          # общие/дефолтные значения
│       ├── values-stage.yaml    # overrides для stage
│       ├── values-prod.yaml     # overrides для prod
│       └── templates/
│           ├── deployment.yaml  # Deployment + env, probes, resources
│           ├── service.yaml     # ClusterIP / NodePort
│           └── ingress.yaml     # Ingress (conditional, по enabled)
└── Dockerfile
```

## Принципы

1. **GitHub Actions** — CI/CD (сборка образа, прогон тестов, пуш в GHCR, обновление Helm values, триггер ArgoCD sync).
2. **Helm chart** — шаблоны Kubernetes (Deployment, Service, Ingress). Все env-переменные через values.
3. **Две ветки/окружения**: `stage` (ручная синхронизация ArgoCD) и `main` → prod (автоматический деплой).
4. **GHCR** (ghcr.io) — registry для Docker-образов.
5. **ArgoCD** — GitOps-оператор, синхронизирует Helm chart из репозитория.

## Этапы настройки CI/CD

### 1. Создать GitHub Actions workflow

Два файла:

- **`ci-docker-publish.yml`** — сборка образа при создании git-тега `v*.*.*`. Тегирует `latest`, `v<version>` и `sha-<commit>`.
- **`docker-ghcr.yml`** — полный пайплайн: тесты → сборка → пуш в GHCR → обновление `values-prod.yaml` → коммит → ArgoCD sync.

Подробные шаблоны см. в [reference-github-actions.md](reference-github-actions.md).

### 2. Создать Helm chart

Шаблоны — в [reference-helm-chart.md](reference-helm-chart.md).

### 3. Создать ветки stage и prod

- **`stage`** — ArgoCD Application `hyranse-backend-stage` использует `values-stage.yaml`. Синхронизация вручную или по расписанию.
- **`main`** → **prod** — ArgoCD Application `hyranse-backend` использует `values-prod.yaml`. Workflow `docker-ghcr.yml` сам обновляет `tag` в `values-prod.yaml` и коммитит.

### 4. Создать ArgoCD Application-манифесты

```
deploy/application.yaml        # prod
deploy/application-stage.yaml  # stage
```

Указывают:
- `source.repoURL` — репозиторий
- `source.targetRevision` — HEAD (или имя ветки)
- `source.path` — `deploy/chart`
- `source.helm.valueFiles` — соответствующий `values-*.yaml`
- `destination.namespace` — `hyranse-backend` или `hyranse-stage`

## Настройка Ingress (опционально)

Ingress настраивается через **поддомены** — каждое окружение на своём поддомене.

| Окружение | Поддомен | Namespace |
|-----------|----------|-----------|
| prod  | `backend.<domain>` | `hyranse-backend` |
| stage | `backend-stage.<domain>` | `hyranse-stage` |

При активации скилла агент запрашивает:

- **Включить Ingress?** (Y/n) — если нет, сервисы доступны только по NodePort
- **Домен для prod** (host): по умолчанию `backend.hyranse.com`
- **Домен для stage** (host): по умолчанию `backend-stage.hyranse.com`
- **Включить TLS через cert-manager?** (Y/n)
- **Имя TLS-секрета**: если да — заполнить

Stage и prod могут иметь независимые настройки TLS.

### Как это работает

```yaml
# prod: backend.hyranse.com → hyranse-backend
# stage: backend-stage.hyranse.com → hyranse-backend-stage
```

Каждый Ingress живёт в своём namespace и указывает на свой сервис. nginx не нужно мержить правила, `rewrite-target` не требуется.

## Health checks и Probes (Kubernetes)

Для каждого приложения настраиваются 3 типа probe.

### 1. Startup probe

Определяет, что приложение полностью запустилось (БД подключена, зависимости готовы). Пока startup не пройдёт — liveness и readiness отключены.

Путь в приложении: `GET /ready` (тот же, что readiness)

### 2. Liveness probe

Проверяет, что процесс жив. Должен быть лёгким и быстрым — без проверки внешних зависимостей.

Путь в приложении: `GET /healthz`

Пример ответа:
```json
{ "status": "ok" }
```

### 3. Readiness probe

Проверяет, что сервис готов принимать трафик. **Должен проверять все связанные ресурсы:**
- PostgreSQL (SELECT 1)
- OpenSearch (ping кластера)
- Redis (если используется)
- Другие API (contactsApiUrl)

Если хотя бы одна зависимость недоступна — readiness возвращает 503.

Путь в приложении: `GET /ready`

Пример ответа при успехе:
```json
{
  "status": "ok",
  "checks": {
    "postgresql": "ok",
    "opensearch": "ok"
  }
}
```

Пример ответа при ошибке:
```json
{
  "status": "error",
  "message": "postgresql unavailable"
}
```

### Реализация в бекенде

Probes должны быть вынесены в **отдельный модуль** `src/health/`:

```
src/health/
├── health.module.ts      # регистрация модуля
├── health.controller.ts  # эндпоинты /healthz и /ready
└── health.service.ts     # проверка зависимостей
```

Референс: `Hyranse_email_plugin/backend/src/health/` — актуальная реализация.

### Настройка в Helm chart

Шаблон `deploy/chart/templates/deployment.yaml` включает conditional-блоки для всех трёх probe (см. [reference-helm-chart.md](reference-helm-chart.md)).

### Значения по умолчанию

| Probe | Path | initialDelaySeconds | periodSeconds | timeoutSeconds | failureThreshold |
|-------|------|---------------------|---------------|----------------|-----------------|
| startup | `/ready` | 5 | 5 | 3 | 36 |
| liveness | `/healthz` | 15 | 60 | 5 | 3 |
| readiness | `/ready` | 5 | 10 | 5 | 3 |

## Prometheus monitoring

### Архитектура

```
Blackbox Exporter  ──scrape──>  /healthz (внешний probe)
       │
       ▼
   Prometheus
       │
       ├── kube-state-metrics ──> метрики pod (restarts, ready)
       ├── blackbox ──> probe_success
       └── Alertmanager ──> Telegram
```

### Blackbox exporter

Устанавливается отдельно в namespace `monitoring`. Шаблон: [reference-prometheus.md](reference-prometheus.md).

### Scrape config (extraScrapeConfigs)

Для каждого сервиса добавляется job blackbox:

```yaml
- job_name: blackbox-<service-name>
  scrape_interval: 1m
  scrape_timeout: 15s
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - http://<service>.<namespace>.svc.cluster.local/healthz
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: prometheus-blackbox-exporter.monitoring.svc.cluster.local:9115
```

### Alerting rules

Для каждого сервиса добавляются правила (в `alerting_rules.yml`):

```yaml
- name: <service-name>-health
  rules:
    - alert: <ServiceName>NotReady
      expr: |
        kube_pod_container_status_ready{
          namespace="<namespace>",
          pod=~"<pod-prefix>-.*"
        } == 0
      for: 2m
      labels:
        severity: critical
        service: <service-name>
      annotations:
        summary: "🚨 Readiness: {{ $labels.pod }}"
        description: "Контейнер {{ $labels.container }} не готов >2min"

    - alert: <ServiceName>HealthFail
      expr: |
        probe_success{
          job="blackbox-<service-name>",
          instance=~".*<service>\\.<namespace>\\.svc\\.cluster\\.local/"
        } == 0
      for: 1m
      labels:
        severity: critical
        service: <service-name>
      annotations:
        summary: "❌ Backend health probe: {{ $labels.instance }}"
        description: "HTTP не 200 или таймаут, раз в минуту scrape"
```

Правила для stage отдельные (c namespace `hyranse-stage`), для prod — `hyranse-backend`.

### Prometheus values

Prometheus устанавливается через Helm-чарт (repo: prometheus-community), с `kubeStateMetrics.enabled: true` для получения метрик pod. Полный шаблон: [reference-prometheus.md](reference-prometheus.md).

## Чего НЕ делать в этом скилле

- Не создавать PostgreSQL, OpenSearch, Redis и другие внешние ресурсы
- Не настраивать cert-manager, ingress-nginx, сам ArgoCD
- Не создавать секреты в Kubernetes
- Не писать бизнес-логику приложения (health-модуль `src/health/` — исключение, он часть деплоя)

## Чеклист при настройке CI/CD

- [ ] Создан Dockerfile в корне проекта
- [ ] Создан health-модуль (`src/health/`) с эндпоинтами `/healthz` и `/ready`
- [ ] `/ready` проверяет PostgreSQL, OpenSearch и другие зависимости
- [ ] Созданы GitHub Actions workflows (сборка + деплой)
- [ ] Создан Helm chart (Chart.yaml + templates)
- [ ] В `deployment.yaml` настроены startup, liveness и readiness probes
- [ ] Настроены values-stage.yaml и values-prod.yaml
- [ ] Созданы ArgoCD Application манифесты для stage и prod
- [ ] Созданы ветки stage и main (prod)
- [ ] Настроены secrets в GitHub: `GITHUB_TOKEN` (авто), `ARGOCD_TOKEN`, `TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID` (опционально)
- [ ] Настроены variables в GitHub: `ARGOCD_BASE_URL` (опционально)
- [ ] Убедиться, что `imagePullSecrets` (ghcr-cred) существует в namespace'ах
- [ ] Если нужен Ingress: указаны домены для stage и prod, решено про TLS
- [ ] Prometheus: добавлен blackbox scrape config и alerting rules для сервиса
- [ ] Prometheus: настроены отдельные alerting rules для stage и prod

## Текущие параметры подключения к ArgoCD

Эти значения должны быть установлены как secrets/variables в GitHub Repository.

| Параметр | Значение |
|----------|----------|
| `ARGOCD_BASE_URL` (vars) | `https://159.195.72.151:30000` |
| `ARGOCD_TOKEN` (secrets) | `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJhcmdvY2QiLCJzdWIiOiJhZG1pbjphcGlLZXkiLCJuYmYiOjE3ODEwOTEyNTEsImlhdCI6MTc4MTA5MTI1MSwianRpIjoiNGJkZTI4ODQtNzMxMC00OTBiLWI5YTItMzJkNTUyMjRiYWY3In0.VeHqNSvm7agk8V0sy1shYOL38vwcpcjvQTfbAGjJNrg` |

> **Важно**: Токен долгоживущий (без срока действия). Хранить в GitHub Secrets, не коммитить в репозиторий.

## Полные шаблоны

- GitHub Actions: [reference-github-actions.md](reference-github-actions.md)
- Helm Chart (values + templates): [reference-helm-chart.md](reference-helm-chart.md)
- ArgoCD Application манифесты: [reference-argo.md](reference-argo.md)
- Prometheus + Blackbox: [reference-prometheus.md](reference-prometheus.md)
