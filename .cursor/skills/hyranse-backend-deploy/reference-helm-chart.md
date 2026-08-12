# Helm Chart — шаблоны

> Замените `<app-name>`, `<repo-owner>`, `<repo-name>`, `<db-name>` на свои значения.
> Референс: `hyranse_backend_main/deploy/chart/`.

## `deploy/chart/Chart.yaml`

```yaml
apiVersion: v2
name: <app-name>
description: <app-name> backend
type: application
version: 0.1.0
appVersion: "1.0"
```

## `deploy/chart/values.yaml`

Общие значения. Переопределяются в `values-stage.yaml` и `values-prod.yaml`.

```yaml
replicaCount: 1
appName: <app-name>

image:
  repository: ghcr.io/<repo-owner>/<repo-name>
  tag: latest
  pullPolicy: Always

imagePullSecrets:
  - name: ghcr-cred

service:
  type: ClusterIP
  port: 80
  targetPort: 8080
  nodePort: 30081

ingress:
  enabled: false
  className: nginx
  host: ""
  path: /
  pathType: Prefix
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
  tls:
    enabled: false
    secretName: ""

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "2"
    memory: 1Gi

probes:
  startup:
    enabled: true
    path: /ready
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 5
    failureThreshold: 36
  readiness:
    enabled: true
    path: /ready
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3
  liveness:
    enabled: true
    path: /healthz
    initialDelaySeconds: 15
    periodSeconds: 60
    timeoutSeconds: 5
    failureThreshold: 3

env:
  port: "8080"
  environment: KUBERNETES
  dbHost: ""
  dbPort: "5432"
  dbUsername: postgresql
  dbName: <db-name>
  opensearchUrl: ""
  opensearchIndex: ""
  contactsApiUrl: ""
  jwtSecret: ""
  xApiKey: ""
  googleClientId: ""
  frontendUrl: ""
  frontendUrls: ""
  landingUrls: ""
  backendUrl: ""
  aiSearchUrl: ""
  brevoApiKey: ""
  mailSender: ""
  responseSecret: ""
  responseSalt: ""
  serviceType: KUBERNETES
  telegramBotToken: ""
  sentryDsn: ""
  sentryEnvironment: ""
  linkedinParserDbHost: ""
  linkedinParserDbPort: "5432"
  linkedinParserDbUsername: postgresql
  linkedinParserDbName: linkedin_db
  linkedinParserDbPassword: ""

dbPasswordSecret:
  name: <app-name>-postgresql
  key: password

opensearchUrlSecret:
  enabled: false
  name: opensearch-credentials
  key: internal-url

appSecret:
  enabled: true
  name: <app-name>-app-secret
  optional: true

linkedinParserDbPasswordSecret:
  enabled: false
  name: hyranse-parser-postgresql
  key: password
```

> **opensearchUrlSecret**: если `enabled: true`, `CANDIDATE_OPENSEARCH_URL` берётся из Secret. Нельзя одновременно задавать literal в `env.opensearchUrl` — k8s отклонит Deployment.

## `deploy/chart/values-stage.yaml`

```yaml
appName: <app-name>-stage

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "2"
    memory: 1Gi

service:
  type: NodePort
  nodePort: 30091

env:
  dbHost: <db-host>
  opensearchIndex: <opensearch-index>
  contactsApiUrl: <contacts-api-url>
  backendUrl: <backend-url>
  frontendUrl: <frontend-url>
  frontendUrls: <frontend-urls>
  landingUrls: <landing-urls>
  aiSearchUrl: <ai-search-url>
  sentryEnvironment: stage

opensearchUrlSecret:
  enabled: true
  name: opensearch-credentials-prod
  key: internal-url

appSecret:
  enabled: false

image:
  tag: latest
```

Stage по умолчанию **без Ingress** (наследует `ingress.enabled: false`). Доступ через NodePort.

## `deploy/chart/values-prod.yaml`

```yaml
appName: <app-name>

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "2"
    memory: 1Gi

service:
  type: NodePort
  nodePort: 30081

ingress:
  enabled: true
  host: <prod-domain>
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
  tls:
    enabled: true
    secretName: <domain>-tls

env:
  dbHost: <db-host>
  opensearchIndex: <opensearch-index>
  contactsApiUrl: <contacts-api-url>
  backendUrl: <backend-url>
  frontendUrl: <frontend-url>
  frontendUrls: <frontend-urls>
  landingUrls: <landing-urls>
  aiSearchUrl: <ai-search-url>
  sentryEnvironment: production

opensearchUrlSecret:
  enabled: true
  name: opensearch-credentials-prod
  key: internal-url

appSecret:
  enabled: true
  name: <app-name>-app-secret
  optional: true

image:
  tag: latest
```

## `deploy/chart/templates/deployment.yaml`

Полный шаблон — копировать из `hyranse_backend_main/deploy/chart/templates/deployment.yaml`.

Ключевые блоки:

```yaml
            - name: CANDIDATE_OPENSEARCH_URL
              {{- if .Values.opensearchUrlSecret.enabled }}
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.opensearchUrlSecret.name }}
                  key: {{ .Values.opensearchUrlSecret.key }}
              {{- else }}
              value: {{ .Values.env.opensearchUrl | quote }}
              {{- end }}
```

```yaml
            {{- if .Values.linkedinParserDbPasswordSecret.enabled }}
            - name: LINKEDIN_PARSER_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.linkedinParserDbPasswordSecret.name }}
                  key: {{ .Values.linkedinParserDbPasswordSecret.key }}
            {{- else if .Values.env.linkedinParserDbPassword }}
            - name: LINKEDIN_PARSER_DB_PASSWORD
              value: {{ .Values.env.linkedinParserDbPassword | quote }}
            {{- end }}
```

Probes (startup → readiness → liveness) — conditional-блоки из `values.probes.*`.

## `deploy/chart/templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.appName }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Values.appName }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      {{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end }}
```

## `deploy/chart/templates/ingress.yaml`

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Values.appName }}
  annotations:
    {{- range $key, $value := .Values.ingress.annotations }}
    {{ $key }}: {{ $value | quote }}
    {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  {{- if .Values.ingress.tls.enabled }}
  tls:
    - hosts:
        - {{ .Values.ingress.host | quote }}
      secretName: {{ .Values.ingress.tls.secretName }}
  {{- end }}
  rules:
    - host: {{ .Values.ingress.host | quote }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: {{ .Values.ingress.pathType }}
            backend:
              service:
                name: {{ .Values.appName }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

Ingress указывает на `{{ .Values.appName }}` напрямую, без отдельных `serviceName` / `servicePort` в values.
