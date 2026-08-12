# GitHub Actions — шаблоны

## `.github/workflows/ci-docker-publish.yml`

Сборка образа по git-тегам `v*.*.*`. Подходит для ручных релизов.

```yaml
name: CI & Docker Publish

on:
  push:
    tags: [ 'v*.*.*' ]

permissions:
  contents: read
  packages: write

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  docker-publish:
    name: Build & Push Docker image
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Read version from package.json
        id: pkg
        shell: bash
        run: echo "version=$(jq -r .version package.json)" >> "$GITHUB_OUTPUT"

      - name: Normalize image name (lowercase)
        id: img
        shell: bash
        run: echo "name=${GITHUB_REPOSITORY,,}" >> "$GITHUB_OUTPUT"

      - name: Setup Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & Push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          platforms: linux/amd64
          tags: |
            ghcr.io/${{ steps.img.outputs.name }}:latest
            ghcr.io/${{ steps.img.outputs.name }}:v${{ steps.pkg.outputs.version }}
            ghcr.io/${{ steps.img.outputs.name }}:sha-${{ github.sha }}
          labels: |
            org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
```

## `.github/workflows/docker-ghcr.yml`

Prod pipeline: тесты → сборка → пуш → обновление `values-prod.yaml` → коммит → ArgoCD sync. Триггер — пуш в `prod`.

> Замените `<ARGO_APP_NAME>` под ваш проект.

```yaml
name: Build and deploy prod

on:
  push:
    branches:
      - prod
    paths:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'
      - 'Dockerfile'
      - 'tsconfig*.json'
      - 'nest-cli.json'
      - 'deploy/**'
      - '.github/workflows/docker-ghcr.yml'
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  HELM_VALUES_PROD: deploy/chart/values-prod.yaml
  ARGO_APP_NAME: <your-app-name>

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: write
      packages: write
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm

      - name: Run tests
        id: run-tests
        run: |
          npm ci
          npm test -- --passWithNoTests

      - name: Send Telegram notification on failure
        if: failure() && steps.run-tests.outcome == 'failure'
        env:
          TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
          TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
        run: |
          if [ -z "$TELEGRAM_BOT_TOKEN" ] || [ -z "$TELEGRAM_CHAT_ID" ]; then
            echo "Skipping Telegram (TELEGRAM_BOT_TOKEN / TELEGRAM_CHAT_ID not set)"
            exit 0
          fi
          curl -sS -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
            -d chat_id="$TELEGRAM_CHAT_ID" \
            --data-urlencode "text=❌ Tests failed for ${{ github.repository }} on branch ${{ github.ref_name }}. Build and push aborted."

      - name: Set lowercase image name
        run: echo "IMAGE_NAME=${GITHUB_REPOSITORY,,}" >> $GITHUB_ENV

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: |
            ghcr.io/${{ env.IMAGE_NAME }}:latest
            ghcr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64
          provenance: false

      - name: Update prod Helm values
        run: |
          sed -i -E "s/^([[:space:]]*tag:[[:space:]]*).*/\1${{ github.sha }}/" "${{ env.HELM_VALUES_PROD }}"

      - name: Commit and push prod image tag
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add "${{ env.HELM_VALUES_PROD }}"
          if git diff --cached --quiet; then
            echo "Helm values already up to date"
            exit 0
          fi
          git commit -m "deploy(prod): update image tag to ${{ github.sha }}"
          git push

      - name: Argo CD sync
        env:
          ARGOCD_TOKEN: ${{ secrets.ARGOCD_TOKEN }}
          ARGOCD_BASE_URL: ${{ vars.ARGOCD_BASE_URL }}
        run: |
          if [ -z "$ARGOCD_TOKEN" ]; then
            echo "ARGOCD_TOKEN secret not configured; skip Argo sync"
            exit 0
          fi
          if [ -z "$ARGOCD_BASE_URL" ]; then
            echo "ARGOCD_BASE_URL variable not configured; skip Argo sync"
            exit 0
          fi
          curl --fail --location --request POST "${ARGOCD_BASE_URL}/api/v1/applications/${{ env.ARGO_APP_NAME }}/sync" \
            --header 'Content-Type: application/json' \
            --cookie "argocd.token=${ARGOCD_TOKEN}" \
            --data '{}'
```

## `.github/workflows/docker-stage.yml`

Stage pipeline: аналогичен prod, но ветка `stage`, теги `stage-latest` / `stage-<sha>`, обновляет `values-stage.yaml`.

```yaml
name: Build and deploy stage image

on:
  push:
    branches:
      - stage
    paths:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'
      - 'Dockerfile'
      - 'tsconfig*.json'
      - 'nest-cli.json'
      - 'deploy/**'
      - '.github/workflows/docker-stage.yml'
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  HELM_VALUES_STAGE: deploy/chart/values-stage.yaml
  ARGO_APP_STAGE: <your-app-name>-stage

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: write
      packages: write
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm

      - name: Run tests
        id: run-tests
        run: |
          npm ci
          npm test -- --passWithNoTests

      - name: Send Telegram notification on failure
        if: failure() && steps.run-tests.outcome == 'failure'
        env:
          TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
          TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
        run: |
          if [ -z "$TELEGRAM_BOT_TOKEN" ] || [ -z "$TELEGRAM_CHAT_ID" ]; then
            echo "Skipping Telegram (TELEGRAM_BOT_TOKEN / TELEGRAM_CHAT_ID not set)"
            exit 0
          fi
          curl -sS -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
            -d chat_id="$TELEGRAM_CHAT_ID" \
            --data-urlencode "text=❌ Tests failed for ${{ github.repository }} on branch ${{ github.ref_name }}. Build aborted."

      - name: Set lowercase image name
        run: echo "IMAGE_NAME=${GITHUB_REPOSITORY,,}" >> $GITHUB_ENV

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: |
            ghcr.io/${{ env.IMAGE_NAME }}:stage-latest
            ghcr.io/${{ env.IMAGE_NAME }}:stage-${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64
          provenance: false

      - name: Update stage Helm values
        run: |
          sed -i -E "s/^([[:space:]]*tag:[[:space:]]*).*/\1stage-${{ github.sha }}/" "${{ env.HELM_VALUES_STAGE }}"

      - name: Commit and push stage image tag
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add "${{ env.HELM_VALUES_STAGE }}"
          if git diff --cached --quiet; then
            echo "Helm values already up to date"
            exit 0
          fi
          git commit -m "deploy(stage): update image tag to stage-${{ github.sha }}"
          git push

      - name: Argo CD sync stage
        env:
          ARGOCD_TOKEN: ${{ secrets.ARGOCD_TOKEN }}
          ARGOCD_BASE_URL: ${{ vars.ARGOCD_BASE_URL }}
        run: |
          if [ -z "$ARGOCD_TOKEN" ]; then
            echo "ARGOCD_TOKEN secret not configured; skip Argo sync"
            exit 0
          fi
          if [ -z "$ARGOCD_BASE_URL" ]; then
            echo "ARGOCD_BASE_URL variable not configured; skip Argo sync"
            exit 0
          fi
          curl --fail --insecure --location --request POST "${ARGOCD_BASE_URL}/api/v1/applications/${{ env.ARGO_APP_STAGE }}/sync" \
            --header 'Content-Type: application/json' \
            --cookie "argocd.token=${ARGOCD_TOKEN}" \
            --data '{}'
```

> `--insecure` в stage workflow нужен, если ArgoCD использует self-signed TLS. Для prod можно убрать, если сертификат валидный.
