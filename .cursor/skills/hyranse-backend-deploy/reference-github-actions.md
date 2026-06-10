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

Полный пайплайн: тесты → сборка → пуш → обновление Helm values → коммит → ArgoCD sync. Триггер — пуш в main (prod).

> Замените `<node-version>`, `<cache-path>`, `<ARGO_APP_NAME>` под ваш проект.

```yaml
name: Build and deploy prod

on:
  push:
    branches:
      - main
    paths:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'
      - 'Dockerfile'
      - 'tsconfig*.json'
      - 'deploy/**'
      - '.github/workflows/docker-ghcr.yml'
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  HELM_VALUES_PROD: deploy/chart/values-prod.yaml
  ARGO_APP_NAME: <your-app-name>     # имя ArgoCD Application

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
          node-version: '<node-version>'  # 20 или 22
          cache: npm
          cache-dependency-path: <cache-path>  # например, backend/package-lock.json

      - name: Run tests
        id: run-tests
        run: |
          cd <subdir>          # если проект не в корне, иначе удалить
          npm ci
          npm test

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
          context: <subdir>            # например, ./backend (или .)
          file: <subdir>/Dockerfile    # например, ./backend/Dockerfile (или ./Dockerfile)
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
          BASE_URL="${ARGOCD_BASE_URL:-http://159.195.72.151:30000}"
          curl --fail --location --request POST "${BASE_URL}/api/v1/applications/${{ env.ARGO_APP_NAME }}/sync" \
            --header 'Content-Type: application/json' \
            --cookie "argocd.token=${ARGOCD_TOKEN}" \
            --data '{}'
```
