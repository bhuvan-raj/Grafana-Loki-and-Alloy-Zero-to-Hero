# Lab 3 — Kubernetes Logging with Grafana Alloy

This lab demonstrates centralized Kubernetes Pod logging.

## Architecture

```text
Kubernetes Cluster
│
├── Application Pod
│       │
│       │ logs
│       ▼
│    Grafana Alloy
│       │
│       │ push
│       ▼
│      Loki
│       │
│       ▼
│    Grafana
│
└── Other Pods
```

## Important

Alloy provides Kubernetes-specific log collection components.

For example:

```text
loki.source.kubernetes
```

can tail Kubernetes Pod logs through the Kubernetes API.

This is different from collecting node filesystem logs.

## Prerequisites

- Kubernetes cluster
- `kubectl`
- Helm
- Loki
- Grafana
- Alloy

## 1. Create a Test Namespace

```bash
kubectl create namespace logging-lab
```

## 2. Deploy a Test Application

Create `log-generator.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-generator
  namespace: logging-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: log-generator
  template:
    metadata:
      labels:
        app: log-generator
    spec:
      containers:
        - name: app
          image: busybox
          command:
            - /bin/sh
            - -c
            - |
              while true; do
                echo "$(date) INFO application is running"
                echo "$(date) ERROR example error message"
                sleep 5
              done
```

Apply:

```bash
kubectl apply -f log-generator.yaml
```

Check:

```bash
kubectl logs -n logging-lab deployment/log-generator
```

## 3. Alloy Pipeline

Conceptually:

```text
Kubernetes Pod
      ↓
loki.source.kubernetes
      ↓
loki.process
      ↓
loki.write
      ↓
Loki
```

A simplified source configuration is:

```alloy
loki.source.kubernetes "pods" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [loki.write.loki.receiver]
}

loki.write "loki" {
  endpoint {
    url = "http://<LOKI_SERVICE>:3100/loki/api/v1/push"
  }
}
```

The discovery and RBAC configuration depends on how Alloy is deployed in the cluster.

## 4. Verify Alloy

```bash
kubectl get pods -n <alloy-namespace>
```

Check Alloy logs:

```bash
kubectl logs -n <alloy-namespace> <alloy-pod>
```

## 5. Verify Loki

```bash
kubectl get pods -n <loki-namespace>
```

Check Loki readiness using the Loki service exposed inside the cluster.

## 6. Query from Grafana

Open:

```text
Explore
```

Select Loki.

Start with a query matching your configured labels, for example:

```logql
{namespace="logging-lab"}
```

Search for errors:

```logql
{namespace="logging-lab"} |= "ERROR"
```

## Important Kubernetes Note

`loki.source.kubernetes` reads Kubernetes Pod logs through the Kubernetes API.

It does **not** collect Kubernetes node logs.

For node-level files such as:

```text
/var/log/...
```

use a file-based collection design with Alloy deployed so it can access the required node filesystem.

## What You Learned

```text
Pod
 ↓
Alloy
 ↓
Loki
 ↓
LogQL
 ↓
Grafana
```
