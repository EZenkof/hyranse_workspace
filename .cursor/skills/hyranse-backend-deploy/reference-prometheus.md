# Prometheus + Blackbox — шаблоны

Источник истины в проекте: `k8s_argo/observability/prometeus/`.

## Blackbox exporter

Устанавливается отдельно через ArgoCD в namespace `monitoring`.

```yaml
# deploy/blackbox-exporter.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus-blackbox-exporter
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-5"
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: prometheus-blackbox-exporter
    targetRevision: 11.10.0
    helm:
      releaseName: prometheus-blackbox-exporter
      values: |
        fullnameOverride: prometheus-blackbox-exporter
        config:
          modules:
            http_2xx:
              prober: http
              timeout: 10s
              http:
                valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
                valid_status_codes: [200]
                method: GET
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Prometheus server

Устанавливается через Helm-чарт `prometheus-community/prometheus`.

### Ключевые параметры values

```yaml
# prometheus:
server:
  replicaCount: 1
  retention: 15d

# Включить kube-state-metrics для метрик pod
kubeStateMetrics:
  enabled: true

# Alertmanager с Telegram
alertmanager:
  enabled: true
  config:
    global:
      resolve_timeout: 5m
    route:
      receiver: telegram-critical
      group_by: ['alertname', 'service']
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 3h
    receivers:
      - name: telegram-critical
        telegram_configs:
          - bot_token: "<telegram-bot-token>"
            chat_id: <telegram-chat-id>
            message: >-
              *{{ .Status | toUpper }}* *{{ .CommonLabels.alertname }}*
              Service: {{ .CommonLabels.service }}
              Instance: {{ .CommonLabels.instance }}
              Namespace: {{ .CommonLabels.namespace }}
              Pod: {{ .CommonLabels.pod }}
              Severity: {{ .CommonLabels.severity }}
              Summary: {{ .CommonAnnotations.summary }}
              Description: {{ .CommonAnnotations.description }}
            parse_mode: "Markdown"
```

### Alerting rules

Добавляются в `serverFiles.alerting_rules.yml`. Для каждого сервиса — своя группа правил.

**Шаблон для одного сервиса:**

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

Правила для каждого окружения — отдельная группа с разными namespace:
- stage: `namespace="<stage-namespace>"`
- prod: `namespace="<prod-namespace>"`

### Scrape config (extraScrapeConfigs)

Для каждого сервиса добавляется blackbox job:

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

## Откуда брать ссылки в проекте

| Компонент | Файл/путь |
|-----------|-----------|
| Prometheus ArgoCD Application | `k8s_argo/observability/prometeus/prometheus.yaml` |
| Blackbox exporter | `k8s_argo/observability/prometeus/blackbox-exporter.yaml` |
| Alerting rules | `prometheus.yaml` → `serverFiles.alerting_rules.yml` |
| Scrape configs | `prometheus.yaml` → `extraScrapeConfigs` |
| Telegram alerts | `prometheus.yaml` → `alertmanager.config.receivers` |
