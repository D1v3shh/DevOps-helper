# Kubernetes Reference

## Table of Contents
1. Core Concepts
2. Deployment
3. Service & Ingress
4. ConfigMap & Secrets
5. Helm Charts
6. Useful kubectl Commands
7. File Structure

---

## 1. Core Concepts

```
Cluster
└── Node (VM/physical machine)
    └── Pod (one or more containers)
        └── Container (your Docker image)

Control Plane manages: Scheduling, Scaling, Self-healing, Rolling updates
```

Key objects:
| Object | Purpose |
|---|---|
| Pod | Smallest deployable unit (wraps containers) |
| Deployment | Manages replica sets, rolling updates |
| Service | Stable network endpoint for pods |
| Ingress | HTTP routing rules (hostname, path) |
| ConfigMap | Non-secret config data |
| Secret | Sensitive config (base64 encoded) |
| PersistentVolume | Storage that outlives pods |

---

## 2. Deployment

```yaml
# k8s/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3                     # Run 3 identical pods
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1           # Keep at least 2 pods running during update
      maxSurge: 1                 # Allow 1 extra pod during rollout
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: ghcr.io/myorg/myapp:v1.2.3   # Always use a specific tag
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
          resources:
            requests:             # Minimum guaranteed resources
              memory: "128Mi"
              cpu: "100m"
            limits:               # Maximum allowed resources
              memory: "256Mi"
              cpu: "500m"
          readinessProbe:         # Pod is ready to receive traffic?
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:          # Pod is alive? Restart if not.
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
```

---

## 3. Service & Ingress

```yaml
# k8s/base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app             # Routes to pods with this label
  ports:
    - port: 80              # Service port
      targetPort: 3000      # Container port
  type: ClusterIP           # Internal only (use LoadBalancer for external)
```

```yaml
# k8s/base/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod   # Auto TLS
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

---

## 4. ConfigMap & Secrets

```yaml
# k8s/base/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  FEATURE_FLAG_NEW_UI: "true"
  config.json: |
    {
      "maxRetries": 3,
      "timeout": 30
    }
```

```yaml
# k8s/base/secret.yaml
# ⚠️ In real projects, use Sealed Secrets or External Secrets Operator
# Never commit plain base64 values to Git
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  database-url: cG9zdGdyZXM6Ly8uLi4=   # base64 encoded
  api-key: c2VjcmV0X2tleQ==
```

---

## 5. Helm Chart

```yaml
# helm/Chart.yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 0.1.0
appVersion: "1.0.0"
```

```yaml
# helm/values.yaml
replicaCount: 3

image:
  repository: ghcr.io/myorg/myapp
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: myapp.example.com

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

env:
  NODE_ENV: production
```

---

## 6. Useful kubectl Commands

```bash
# View resources
kubectl get pods -n my-namespace
kubectl get deployments
kubectl describe pod my-pod-name

# Logs
kubectl logs -f deployment/my-app         # Follow logs
kubectl logs my-pod --previous            # Logs from crashed container

# Exec into pod
kubectl exec -it my-pod -- /bin/sh

# Apply manifests
kubectl apply -f k8s/base/
kubectl apply -k k8s/overlays/staging/    # Kustomize

# Rollout
kubectl rollout status deployment/my-app
kubectl rollout undo deployment/my-app    # Rollback

# Scale
kubectl scale deployment my-app --replicas=5

# Port forward (local debugging)
kubectl port-forward svc/my-app 8080:80
```

---

## 7. File Structure

```
k8s/
├── base/                    # Shared across all environments
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml   # Lists all base resources
└── overlays/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patch-replicas.yaml    # Override replicas=1 in staging
    └── production/
        ├── kustomization.yaml
        └── patch-replicas.yaml    # Override replicas=5 in prod

helm/                        # Alternative: use Helm instead of kustomize
├── Chart.yaml
├── values.yaml
├── values-staging.yaml      # Environment overrides
├── values-production.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── _helpers.tpl         # Reusable template snippets
```