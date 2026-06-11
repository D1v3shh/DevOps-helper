# GitOps & Deployment Strategies Reference

## Table of Contents
1. GitOps Principles
2. ArgoCD Setup
3. Deployment Strategies
4. File Structure

---

## 1. GitOps Principles

**GitOps** = Git is the single source of truth for infrastructure and application state.

```
Developer pushes → Git repo updated → GitOps tool detects diff
                                              ↓
                                   Syncs cluster to match Git state
```

Benefits: audit trail, easy rollback (revert commit), pull-based (more secure than push).

---

## 2. ArgoCD Application

```yaml
# gitops/apps/my-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/my-app
    targetRevision: main
    path: k8s/overlays/production    # Where manifests live in the repo

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true        # Delete resources removed from Git
      selfHeal: true     # Revert manual cluster changes
    syncOptions:
      - CreateNamespace=true
```

---

## 3. Deployment Strategies

### Rolling Update (default)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
```
Gradually replaces old pods. Zero downtime. Good default.

### Blue-Green
Run two identical environments. Switch traffic instantly.
```
Blue (current v1) ←── traffic
Green (new v2)    ←── traffic  (after switch)
```

### Canary
Send 5% of traffic to new version. Increase if metrics look good.
```yaml
# With Argo Rollouts
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
```

---

## 4. File Structure

```
gitops/
├── apps/                    # ArgoCD Application definitions
│   ├── my-app.yaml
│   └── infrastructure.yaml
└── clusters/
    ├── staging/
    └── production/
```