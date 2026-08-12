# ArgoCD Application — шаблоны

> Замените `<repo-url>`, `<app-name>`, `<prod-branch>`, `<stage-branch>`, namespace'ы.

ArgoCD читает chart из **веток `prod` и `stage`**, не из default branch. CI коммитит image tag в values на соответствующую ветку, ArgoCD подхватывает изменения.

## `deploy/application.yaml` (prod)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  source:
    repoURL: <repo-url>
    targetRevision: prod
    path: deploy/chart
    helm:
      valueFiles:
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: <prod-namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## `deploy/application-stage.yaml` (stage)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>-stage
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  source:
    repoURL: <repo-url>
    targetRevision: stage
    path: deploy/chart
    helm:
      valueFiles:
        - values-stage.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: <stage-namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Пример для hyranse_backend_main

| Поле | prod | stage |
|------|------|-------|
| `metadata.name` | `hyranse-backend` | `hyranse-backend-stage` |
| `targetRevision` | `prod` | `stage` |
| `destination.namespace` | `hyranse-backend` | `hyranse-stage` |
| `repoURL` | `https://github.com/EZenkof/hyranse_backend_main.git` | тот же |
