# ArgoCD Application — шаблоны

> Замените `<repo-url>`, `<app-name>`, `<namespace>` на свои значения.

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
    targetRevision: HEAD
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
    targetRevision: HEAD
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
