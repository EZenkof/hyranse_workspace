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

## Sentry projects (DSN)

| Сервис | Project | DSN location |
|--------|---------|--------------|
| Backend | отдельный project | `values-stage.yaml`, `values-prod.yaml` |
| Frontend | `javascript-react` | `publish-image.yml`, `.env.local` (local only) |

DSN — публичный client key, допустим в Helm values. `SENTRY_AUTH_TOKEN` — только secrets.

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
