# Sentry — расследование ошибок

## Workflow

### 1. Сбор контекста

Из issue или от пользователя:

- Issue URL или error message
- Environment (`stage` / `production` / `development`)
- Release (git SHA)
- Service (`backend` / `frontend` / `ai-search`)
- Frequency (новая / регрессия / постоянная)

### 2. Sentry UI

| Поле | Что смотреть |
|------|--------------|
| Environment | stage vs production — разные deploys |
| Release | SHA → `git show <sha>` / `git log --oneline <sha>` |
| Breadcrumbs | действия пользователя до ошибки |
| Tags | `path`, `method`, `microfrontend`, `status` |
| Replay | frontend only — session replay с masked text |
| Stack trace | readable если source maps uploaded для release |

### 3. Корреляция с git

```bash
git log --oneline -1 <release-sha>
git diff <release-sha>^ <release-sha> -- <suspected-file>
```

Проверить: issue появился после конкретного deploy?

### 4. Локальное воспроизведение

Backend:

```bash
SENTRY_DSN=<dsn> SENTRY_ENVIRONMENT=development npm run start:dev
```

Frontend:

```bash
REACT_APP_SENTRY_DSN=<dsn> REACT_APP_SENTRY_ENVIRONMENT=development npm start
```

Для prod-only bugs: воспроизвести на stage с тем же code path.

### 5. Фикс

- Root cause fix в коде
- Если шум — добавить в `ignoreErrors` / `beforeSend` / interceptor filter
- Не resolve issue в UI без deploy фикса

### 6. Verify

- Deploy на stage
- Проверить что error rate падает в Sentry
- Regression test если возможно

## CLI

```bash
npx sentry-cli login
npx sentry-cli issues list --org hyranse-go --project javascript-react
npx sentry-cli issues show <issue-id> --org hyranse-go
npx sentry-cli releases list --org hyranse-go
npx sentry-cli releases info <sha> --org hyranse-go
```

`.sentryclirc` в frontend: `url=https://de.sentry.io/`, `org=hyranse-go`, `project=javascript-react`.

Для backend issues — CLI с project из [reference-hyranse.md](reference-hyranse.md#sentry-projects-map):

```bash
npx sentry-cli issues list --org hyranse-go --project hyranse-billing-backend
npx sentry-cli issues list --org hyranse-go --project hyranse-parser
```

Parser: фильтровать в UI по tag `worker` / `shard`, не по отдельным projects.

## Типичные паттерны Hyranse

| Симптом | Вероятная причина | Где смотреть |
|---------|-------------------|--------------|
| Minified stack | source maps не uploaded для release | CI logs, `sentry-upload.sh` |
| Дубль issue | interceptor + Sentry module | `error-logging.interceptor.ts` |
| Шум Network Error | фильтр не сработал | `beforeSend` в instrument/sentry.ts |
| MFE error без context | capture без tags | `captureAiSearchError` |
| Error только prod | env-specific config / data | compare stage vs prod values |
| 403/404 в Sentry (MFE) | фильтр status не применён | `skippedStatuses` в ai-search |

## Двойная отправка (backend)

`ErrorLoggingInterceptor` шлёт в Sentry + Telegram. При добавлении нового capture path проверить:

- Не дублируется ли с автоматическим `@sentry/nestjs` handler
- Expected business errors не попадают в capture

## Privacy checklist

- [ ] Нет PII в tags/extra/breadcrumbs
- [ ] Auth headers stripped в `beforeSend`
- [ ] Replay: `maskAllText: true`, `blockAllMedia: true`
- [ ] Source maps удалены из production image после upload
