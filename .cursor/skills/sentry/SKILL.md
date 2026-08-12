---
name: sentry
description: >-
  Configure, debug, and investigate Sentry in Hyranse projects (NestJS backend,
  React frontend, MFE). Use when working with Sentry errors, DSN, source maps,
  releases, beforeSend filters, replay, traces, or production error investigation.
disable-model-invocation: true
---

# Sentry — Hyranse

Источник истины по интеграции: код в `hyranse_backend_main`, `hyranse_frontend_main`, `hyranse_ai_search_engine`.

**Org:** `hyranse-go` · **Region:** `https://de.sentry.io/`

## Архитектура

| Компонент | SDK | Init | Env vars |
|-----------|-----|------|----------|
| `hyranse_backend_main` | `@sentry/nestjs` | `src/instrument.ts` (первый import в `main.ts`) | `SENTRY_DSN`, `SENTRY_ENVIRONMENT` |
| `hyranse_frontend_main` | `@sentry/react` | `src/sentry.ts` | `REACT_APP_SENTRY_*`, `SENTRY_AUTH_TOKEN` (CI only) |
| `hyranse_ai_search_engine` (MFE) | `@sentry/react` | host app init | capture через `captureAiSearchError` |

Backend и frontend — разные DSN/projects. Карта projects: [reference-hyranse.md](reference-hyranse.md#sentry-projects-map).

## Sentry projects — правила

**Отдельный Sentry project** — для каждого deployable backend (свой Helm chart / Argo Application / lifecycle).

**Не отдельный project:**

| Случай | Решение |
|--------|---------|
| Parser shards (`public-profile-0`, `photo-1`, …) | один `hyranse-parser`, tags: `worker`, `shard`, `deployment` |
| stage + prod | один project, разные `environment` |
| MFE (billing, ai-search, cv-editor, …) | host project `javascript-react` + tag `microfrontend` |
| UI kit, wiki | не нужен — нет runtime |

**Правило:** `отдельный Sentry project ↔ отдельный backend repo/chart`, но не «один project на pod».

**Зачем отдельные backend projects:**

- alerts без кросс-шума (parser spike не алертит billing)
- разные `ignoreErrors` / `beforeSend` per service
- разный sampling (parser high volume → `0.05`, billing → `0.1`)
- event quotas не съедаются одним шумным сервисом
- ownership: issue в project = конкретный сервис

**Не делать** один mega `hyranse-backends` + tag `service` — parser засыпает billing issues, alerts бесполезны.

**Auth:** один `SENTRY_AUTH_TOKEN` в GitHub Secrets (org scope), разные `project` в `.sentryclirc` / CLI per repo.

## Принципы

1. **Init до всего остального** — backend: `import './instrument'` первым в `main.ts`; frontend: `sentry.ts` до React root.
2. **Enabled только когда есть DSN** — backend: `if (dsn) Sentry.init(...)`; frontend: `enabled: isProduction && Boolean(dsn)`.
3. **Release = git SHA** — `REACT_APP_SENTRY_RELEASE=${{ github.sha }}`, не `latest`.
4. **Фильтровать шум в SDK** — не resolve в UI без фикса кода.
5. **Secrets** — `SENTRY_AUTH_TOKEN` только в GitHub Secrets, никогда в repo.
6. **MFE** — не вызывать `Sentry.init` в microfrontend; только `captureException` через client host app.

## Sampling (prod/stage)

| Параметр | Prod/Stage | Dev |
|----------|------------|-----|
| `tracesSampleRate` | `0.1` | `1.0` |
| `replaysSessionSampleRate` | `0.1` | `0` |
| `replaysOnErrorSampleRate` | `1.0` | `0` |

Не ставить `tracesSampleRate: 1.0` в prod без явной причины.

## Фильтрация шума

Shared config (host): `src/observability/sentry-filters.ts` — `SKIPPED_HTTP_STATUSES`, `SKIPPED_ERROR_MESSAGES`, `shouldCaptureHttpStatus`, `shouldCaptureErrorMessage`.

Host interceptor (`error-response.interceptor.ts`) и `sentry.ts` `beforeSend` используют этот модуль. MFE capture helpers дублируют `skippedStatuses` локально (отдельный repo).

### Игнорировать

| Категория | Примеры |
|-----------|---------|
| Сеть | `Network Error`, `ECONNRESET`, `timeout exceeded` |
| Бизнес | `Forbidden resource`, `User not found` |
| Browser | `ResizeObserver loop`, `Loading chunk` |
| HTTP (MFE) | 401, 403, 404 |

### beforeSend

- Возвращать `null` для filtered events.
- Frontend: удалять `Authorization`, `Cookie`, `X-Refresh-Token` из `event.request.headers`.
- Backend: `ignoreErrors` + фильтр в `ErrorLoggingInterceptor`.

## Теги при captureException

```typescript
Sentry.captureException(error, {
  tags: {
    path: request.url,
    method: request.method,
    microfrontend: 'aiSearch',
    status: context?.status,
  },
});
```

Tags — для фильтрации в UI. Не логировать PII в tags/extra.

## Source maps (frontend)

```
build → sentry-cli releases new → inject → upload → finalize → deploys → delete *.map
```

Скрипт: `hyranse_frontend_main/scripts/sentry-upload.sh`. Без `SENTRY_AUTH_TOKEN` или release — skip (exit 0).

## Новый сервис — чеклист

- [ ] Создать Sentry project по naming: `hyranse-<service>-backend` (см. карту в reference-hyranse.md)
- [ ] Не создавать project на shard/stage/MFE
- [ ] Init SDK с `environment` и conditional `enabled`
- [ ] `tracesSampleRate: 0.1` для prod/stage
- [ ] `beforeSend` / `ignoreErrors` для expected errors
- [ ] Strip auth headers в `beforeSend` (frontend)
- [ ] Release = git SHA в CI
- [ ] Source maps upload + delete `.map` из production image
- [ ] Env vars в Helm values / GitHub Actions
- [ ] `SENTRY_AUTH_TOKEN` в GitHub Secrets

## Расследование ошибки

```
1. Issue URL или message + environment + release
2. Sentry UI: environment, release (SHA), breadcrumbs, tags, replay (frontend)
3. git log --oneline <release-sha> → diff deploy
4. Локально воспроизвести с тем же environment
5. Фикс → deploy → проверить regression
6. Шум → добавить фильтр в SDK, не resolve без фикса
```

CLI:

```bash
npx sentry-cli issues list --org hyranse-go --project javascript-react
npx sentry-cli releases list --org hyranse-go
```

## Анти-паттерны

| ❌ | ✅ |
|----|-----|
| `Sentry.init` в каждом MFE | один init в host, capture в MFE |
| `captureException` для 404/403 | фильтр по status / `ignoreErrors` |
| Source maps в production image | upload + delete |
| Auth token в repo | GitHub Secrets |
| Resolve issue без фикса | fix → deploy |
| `captureMessage(JSON.stringify(user))` | никогда — PII |
| Project на parser shard | один `hyranse-parser` + tags |
| Project на stage/prod | `environment`, не project |
| Project на каждый MFE | tag `microfrontend` в `javascript-react` |
| Один mega backend project | отдельный project per deployable backend |
| Расширять `ignoreErrors` без покрытия | ErrorBoundary + capture helpers + status skip |

## Frontend coverage (host + MFE)

Улучшать **покрытие**, не раздувать ignore list.

### Host (`hyranse_frontend_main`)

| Механика | Файл | Статус |
|----------|------|--------|
| `Sentry.init` | `src/sentry.ts` | ✅ |
| Global `ErrorBoundary` | `src/index.tsx` | ✅ |
| Axios interceptor → Sentry | `src/interceptors/error-response.interceptor.ts` | ✅ skip 401/403/404 + messages |
| Shared filters | `src/observability/sentry-filters.ts` | ✅ |
| MFE `ErrorBoundary` billing | `billing-remote.page.tsx` | ✅ |
| MFE `ErrorBoundary` ai-search | `ai-search-remote.page.tsx` | ✅ |
| MFE `ErrorBoundary` cv-editor | `cv-editor-remote.page.tsx` | ✅ |
| MFE `ErrorBoundary` api-portal | `api-portal-remote.page.tsx` | ✅ |

Interceptor: Telegram/toast по-прежнему для большинства ошибок; **Sentry** только если `shouldCaptureHttpStatus` + `shouldCaptureErrorMessage`.

### MFE capture helpers

Паттерн: `frontend/src/observability/sentry.ts` — `captureXError`, проверка `Sentry.getClient()`, skip 401/403/404, tag `microfrontend`.

| MFE | Helper | API layer |
|-----|--------|-----------|
| billing | `captureBillingError` | `billing.service.ts` |
| ai-search | `captureAiSearchError` | `api-client.ts` |
| cv-editor | `captureCvEditorError` | `cv.service.ts` |
| api-portal | `captureApiPortalError` | `public-api.service.ts` |

MFE **не** вызывают `Sentry.init` — только capture через client host app.

### ErrorBoundary на host

```tsx
<Sentry.ErrorBoundary
  fallback={<div>… temporarily unavailable…</div>}
  beforeCapture={(scope) => scope.setTag('microfrontend', 'cvEditor')}
>
  <Suspense>…</Suspense>
</Sentry.ErrorBoundary>
```

Ловит React render crashes в remote; API errors — через capture helper в MFE + host interceptor.

## Дополнительно

- Пути, Helm, CI per repo: [reference-hyranse.md](reference-hyranse.md)
- Workflow расследования и CLI: [reference-investigation.md](reference-investigation.md)
