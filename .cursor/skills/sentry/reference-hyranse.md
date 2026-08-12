# Sentry — пути и конфигурация Hyranse

## hyranse_backend_main

| Файл | Назначение |
|------|------------|
| `src/instrument.ts` | `Sentry.init` |
| `src/main.ts` | `import './instrument'` первым |
| `src/common/interceptors/error-logging.interceptor.ts` | `captureException` + Telegram |
| `deploy/chart/values.yaml` | `env.sentryDsn`, `env.sentryEnvironment` (пустые default) |
| `deploy/chart/values-stage.yaml` | DSN + `sentryEnvironment: stage` |
| `deploy/chart/values-prod.yaml` | DSN + `sentryEnvironment: production` |
| `deploy/chart/templates/deployment.yaml` | `SENTRY_DSN`, `SENTRY_ENVIRONMENT` env vars |

Env vars в pod:

```
SENTRY_DSN
SENTRY_ENVIRONMENT
```

`instrument.ts` читает также `ENVIRONMENT` как fallback.

## hyranse_frontend_main

| Файл | Назначение |
|------|------------|
| `src/sentry.ts` | `Sentry.init` + integrations |
| `src/index.tsx` | import sentry до React |
| `scripts/sentry-upload.sh` | upload source maps |
| `.sentryclirc` | org `hyranse-go`, project `javascript-react` |
| `.github/workflows/publish-image.yml` | build args + secrets |

Env vars:

```
REACT_APP_SENTRY_DSN
REACT_APP_SENTRY_RELEASE      # git SHA
REACT_APP_SENTRY_ENVIRONMENT  # stage / production / development
SENTRY_AUTH_TOKEN             # CI only, GitHub Secrets
```

Build flow в `publish-image.yml`:

1. Set `REACT_APP_SENTRY_*` per branch (stage vs prod)
2. Docker build с build args
3. `sentry-upload.sh` после build

## hyranse_ai_search_engine (MFE)

| Файл | Назначение |
|------|------------|
| `frontend/src/observability/sentry.ts` | `captureAiSearchError`, export `Sentry` |
| `frontend/scripts/sentry-upload.sh` | source maps (аналог frontend) |
| `frontend/.sentryclirc` | org/project |

MFE не инициализирует Sentry — проверка `Sentry.getClient()` перед capture.

`captureAiSearchError` пропускает status 401, 403, 404.

Tags: `microfrontend: aiSearch`, `url`, `status`, `method`.

## Sentry projects map

Org: `hyranse-go` · Region: `https://de.sentry.io/`

| Sentry project | Repo / сервис | Статус | DSN / config location |
|----------------|---------------|--------|------------------------|
| `javascript-react` | `hyranse_frontend_main` + все MFE errors | ✅ active | `.sentryclirc`, `publish-image.yml` |
| `hyranse-backend` | `hyranse_backend_main` | ✅ active | `deploy/chart/values-{stage,prod}.yaml` |
| `hyranse-billing-backend` | `hyranse_billing` backend | ✅ active | Helm values |
| `hyranse-common-backend` | `common_backend` | ✅ active | Helm values |
| `hyranse-parser` | `nodejs_parser_v2` (все workers/shards) | ✅ active | Helm values + tags `worker`, `shard` |
| `hyranse-ai-search-backend` | `hyranse_ai_search_engine` backend | ✅ active | Helm values |
| `hyranse-cv-editor-backend` | `hyranse_cv_editor` backend | ✅ active | Helm values |
| `hyranse-public-api-backend` | `Hyranse_public_api` backend | ✅ active | Helm values |
| `hyranse-email-service-backend` | `hyranse_email_service` backend | ✅ active | env `SENTRY_DSN` (no Helm yet) |
| `hyranse-email-plugin-backend` | `Hyranse_email_plugin` backend | ✅ active | Helm values |
| `hyranse-search-engine-backend` | `Hyranse_search_engine` | ✅ active | Helm values |

### Не создавать отдельные projects

| Что | Почему | Альтернатива |
|-----|--------|--------------|
| Parser shard (`hyranse-parser-public-profile-0`, …) | один codebase, 10+ deployments | `hyranse-parser` + `tags.worker`, `tags.shard` |
| stage / production | дублирование projects | `SENTRY_ENVIRONMENT` / `REACT_APP_SENTRY_ENVIRONMENT` |
| MFE (`billing`, `aiSearch`, `cvEditor`, `apiPortal`) | init только в host | `javascript-react` + `tags.microfrontend` |
| `hyranse_ui_kit`, `hyranse_wiki` | нет runtime | — |

### Naming convention

```
hyranse-<service>-backend     # NestJS/Node backend
javascript-react              # host frontend (+ MFE errors via tags)
```

`<service>` = короткое имя из repo/chart: `billing`, `parser`, `common`, `ai-search`, `cv-editor`, `public-api`, `email-service`, `email-plugin`, `search-engine`.

### Per-project alert defaults

| Project | Priority | `tracesSampleRate` (prod) | Alert |
|---------|----------|---------------------------|-------|
| `hyranse-billing-backend` | critical | `0.1` | new issue in production |
| `hyranse-parser` | high volume | `0.05` | error rate spike per `worker` tag |
| `hyranse-common-backend` | high | `0.1` | new issue in production |
| остальные backends | normal | `0.1` | new issue in production |
| `javascript-react` | normal | `0.1` | new issue in production |

### Parser worker tags (пример)

```typescript
Sentry.captureException(error, {
  tags: {
    worker: process.env.WORKER_NAME,
    shard: process.env.SHARD_INDEX,
    deployment: process.env.DEPLOYMENT_NAME,
  },
});
```

### MFE microfrontend tags

| MFE | tag `microfrontend` | capture helper | ErrorBoundary on host |
|-----|---------------------|----------------|----------------------|
| ai-search | `aiSearch` | `captureAiSearchError` | ✅ |
| billing | `billing` | `captureBillingError` | ✅ |
| cv-editor | `cvEditor` | 🔲 planned | 🔲 planned |
| api-portal | `apiPortal` | 🔲 planned | 🔲 planned |

DSN — публичный client key, допустим в Helm values. `SENTRY_AUTH_TOKEN` — только GitHub Secrets (один token, org scope).

## Frontend init highlights (`sentry.ts`)

Integrations:

- `browserTracingIntegration()`
- `replayIntegration({ maskAllText: true, blockAllMedia: true })`

`tracePropagationTargets`: `app.hyranse.com`, `stage-app.hyranse.com`, `api.hyranse.com`, `stage-api.hyranse.com`, `hyranse.com`, `localhost`

`denyUrls`: browser extensions, `chrome://`

## Backend init highlights (`instrument.ts`)

- `attachStacktrace: true`
- `beforeSend`: filter `Network Error`, `ECONNRESET`
- `ignoreErrors`: `Forbidden resource`, `User not found`

## Helm template snippet

```yaml
{{- if .Values.env.sentryDsn }}
- name: SENTRY_DSN
  value: {{ .Values.env.sentryDsn | quote }}
{{- end }}
{{- if .Values.env.sentryEnvironment }}
- name: SENTRY_ENVIRONMENT
  value: {{ .Values.env.sentryEnvironment | quote }}
{{- end }}
```

## Source maps upload (`sentry-upload.sh`)

```sh
npx sentry-cli releases new "$REACT_APP_SENTRY_RELEASE"
npx sentry-cli sourcemaps inject ./build/static/js
npx sentry-cli sourcemaps upload ./build/static/js --release "$REACT_APP_SENTRY_RELEASE"
npx sentry-cli releases finalize "$REACT_APP_SENTRY_RELEASE"
npx sentry-cli releases deploys "$REACT_APP_SENTRY_RELEASE" new -e "$REACT_APP_SENTRY_ENVIRONMENT"
find ./build/static/js -name '*.map' -delete
```

Skip если нет `SENTRY_AUTH_TOKEN` или `REACT_APP_SENTRY_RELEASE`.
