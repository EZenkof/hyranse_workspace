# Helm Chart — шаблоны

> Замените `<app-name>`, `<repo-owner>`, `<db-name>` на свои значения.

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

Общие/дефолтные значения. Переопределяются в `values-stage.yaml` и `values-prod.yaml`.

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
  rewriteTarget: ""
  serviceName: ""
  servicePort: 80
  annotations: {}
  tls:
    enabled: false
    secretName: ""

resources:
  requests:
    cpu: 100m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi

probes:
  startup:
    enabled: true
    path: /ready
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 36
  liveness:
    enabled: true
    path: /healthz
    initialDelaySeconds: 15
    periodSeconds: 60
    timeoutSeconds: 5
    failureThreshold: 3
  readiness:
    enabled: true
    path: /ready
    initialDelaySeconds: 5
    periodSeconds: 10
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
  opensearchIndex: <opensearch-index>
  contactsApiUrl: ""

dbPasswordSecret:
  name: <app-name>-postgresql
  key: password

appSecret:
  enabled: true
  name: <app-name>-app-secret
  optional: true
```

## `deploy/chart/values-stage.yaml`

```yaml
appName: <app-name>-stage

service:
  type: NodePort
  nodePort: 30091

ingress:
  enabled: true
  host: <stage-domain>             # например, backend-stage.hyranse.com
  path: /
  pathType: Prefix
  rewriteTarget: ""
  serviceName: <app-name>-stage
  servicePort: 80
  annotations: {}
  tls:
    enabled: false

env:
  dbHost: <db-host>
  opensearchUrl: <opensearch-url>
  contactsApiUrl: <contacts-api-url>

image:
  tag: latest
```

## `deploy/chart/values-prod.yaml`

```yaml
appName: <app-name>

service:
  type: NodePort
  nodePort: 30081

ingress:
  enabled: true
  host: <prod-domain>              # например, backend.hyranse.com
  path: /
  pathType: Prefix
  rewriteTarget: ""
  serviceName: <app-name>
  servicePort: 80
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # опционально
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
  tls:
    enabled: true                  # true если есть cert-manager
    secretName: <domain>-tls

env:
  dbHost: <db-host>
  opensearchUrl: <opensearch-url>
  contactsApiUrl: <contacts-api-url>

image:
  tag: latest
```

env:
  dbHost: <db-host>
  opensearchUrl: <opensearch-url>
  contactsApiUrl: <contacts-api-url>

image:
  tag: latest
```

## `deploy/chart/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}
spec:
  replicas: {{ .Values.replicaCount }}
  revisionHistoryLimit: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Values.appName }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            - name: PORT
              value: {{ .Values.env.port | quote }}
            - name: ENVIRONMENT
              value: {{ .Values.env.environment | quote }}
            - name: DB_HOST
              value: {{ .Values.env.dbHost | quote }}
            - name: DB_PORT
              value: {{ .Values.env.dbPort | quote }}
            - name: DB_USERNAME
              value: {{ .Values.env.dbUsername | quote }}
            - name: DB_NAME
              value: {{ .Values.env.dbName | quote }}
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.dbPasswordSecret.name }}
                  key: {{ .Values.dbPasswordSecret.key }}
            {{ if .Values.env.opensearchUrl }}
            - name: CANDIDATE_OPENSEARCH_URL
              value: {{ .Values.env.opensearchUrl | quote }}
            - name: CANDIDATE_OPENSEARCH_INDEX
              value: {{ .Values.env.opensearchIndex | quote }}
            {{ end }}
            {{ if .Values.env.contactsApiUrl }}
            - name: CONTACTS_API_URL
              value: {{ .Values.env.contactsApiUrl | quote }}
            {{ end }}
          {{- if .Values.appSecret.enabled }}
          envFrom:
            - secretRef:
                name: {{ .Values.appSecret.name }}
                optional: {{ .Values.appSecret.optional }}
          {{- end }}
          {{- if .Values.probes.startup.enabled }}
          startupProbe:
            httpGet:
              path: {{ .Values.probes.startup.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: {{ .Values.probes.startup.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.startup.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.startup.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.startup.failureThreshold }}
          {{- end }}
          {{- if .Values.probes.liveness.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.liveness.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.liveness.failureThreshold }}
          {{- end }}
          {{- if .Values.probes.readiness.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.readiness.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.readiness.failureThreshold }}
          {{- end }}
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
```

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
    {{- if .Values.ingress.rewriteTarget }}
    nginx.ingress.kubernetes.io/rewrite-target: {{ .Values.ingress.rewriteTarget | quote }}
    {{- end }}
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
                name: {{ .Values.ingress.serviceName }}
                port:
                  number: {{ .Values.ingress.servicePort }}
{{- end }}
```
