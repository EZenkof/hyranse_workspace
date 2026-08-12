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

Один Sentry project = один deployable artifact. Backend и frontend — разные DSN/projects.

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

- [ ] Выбрать/создать Sentry project (отдельный DSN)
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

## Дополнительно

- Пути, Helm, CI per repo: [reference-hyranse.md](reference-hyranse.md)
- Workflow расследования и CLI: [reference-investigation.md](reference-investigation.md)
