
# Loki + Grafana Alloy on Kubernetes

## 🎯 Lab Objective

In this lab, we will build a complete Kubernetes logging pipeline using:

* Kubernetes
* Grafana Alloy
* Grafana Loki
* Grafana

The final architecture will be:

```text
                    Kubernetes Cluster
┌───────────────────────────────────────────────────────────┐
│                                                           │
│   ┌───────────────────┐                                   │
│   │  Application Pod  │                                   │
│   │                   │                                   │
│   │  nginx / app      │                                   │
│   └─────────┬─────────┘                                   │
│             │                                             │
│             │ Container logs                              │
│             ▼                                             │
│   ┌──────────────────────────┐                            │
│   │      Grafana Alloy       │                            │
│   │                          │                            │
│   │ discovery.kubernetes     │                            │
│   │          ↓               │                            │
│   │ loki.source.kubernetes   │                            │
│   │          ↓               │                            │
│   │ loki.write               │                            │
│   └────────────┬─────────────┘                            │
│                │                                          │
│                │ HTTP Push                               │
│                ▼                                          │
│   ┌──────────────────────────┐                            │
│   │       Grafana Loki       │                            │
│   │                          │                            │
│   │  Log storage + querying  │                            │
│   └────────────┬─────────────┘                            │
│                │                                          │
└────────────────┼──────────────────────────────────────────┘
                 │
                 ▼
        ┌─────────────────┐
        │     Grafana     │
        │     Explore     │
        └─────────────────┘
```

### Alloy pipeline

```text
discovery.kubernetes
        ↓
loki.source.kubernetes
        ↓
loki.write
        ↓
       Loki
```

---

# Prerequisites

Students should have:

* Kubernetes cluster
* `kubectl`
* Helm 3
* Basic Kubernetes knowledge
* Basic Loki/LogQL knowledge

Verify:

```bash
kubectl cluster-info
```

```bash
helm version
```

---

# Part 1 — Deploy Loki

## 1. Create the Loki namespace

```bash
kubectl create namespace loki
```

Verify:

```bash
kubectl get namespace loki
```

---

## 2. Add the Grafana Community Helm repository

The Loki Helm chart is currently maintained in the Grafana Community Helm repository. ([Grafana Labs][3])

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
```

Update the repository:

```bash
helm repo update
```

Verify:

```bash
helm search repo grafana-community/loki
```

---

# 3. Create Loki values

Create a working directory:

```bash
mkdir loki-alloy-lab
cd loki-alloy-lab
```

Create:

```bash
vim loki-values.yaml
```

Use:

```yaml
deploymentMode: Monolithic

loki:
  commonConfig:
    replication_factor: 1

  schemaConfig:
    configs:
      - from: "2024-04-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

  storage:
    type: filesystem

  limits_config:
    allow_structured_metadata: true
    volume_enabled: true

singleBinary:
  replicas: 1

# Disable other deployment modes
backend:
  replicas: 0

read:
  replicas: 0

write:
  replicas: 0

ingester:
  replicas: 0

querier:
  replicas: 0

queryFrontend:
  replicas: 0

queryScheduler:
  replicas: 0

distributor:
  replicas: 0

compactor:
  replicas: 0

indexGateway:
  replicas: 0

bloomPlanner:
  replicas: 0

bloomBuilder:
  replicas: 0

bloomGateway:
  replicas: 0

# We don't need the gateway for this lab
gateway:
  enabled: false

chunksCache:
  enabled: false

resultsCache:
  enabled: false

test:
  enabled: false
```

### Why are we disabling the gateway?

For this lab we need to understand:

```text
Alloy
  ↓
Loki Service
  ↓
Loki
```

rather than introducing the additional NGINX gateway layer.

The Loki chart normally installs a gateway, and Grafana recommends using it when it is enabled. However, the chart also supports bypassing it. ([Grafana Labs][4])

For **this educational lab**, we explicitly disable it:

```yaml
gateway:
  enabled: false
```

This keeps the architecture simple.

---

# 4. Install Loki

Run:

```bash
helm install loki \
  grafana-community/loki \
  -f loki-values.yaml \
  -n loki
```

Wait for Loki:

```bash
kubectl get pods -n loki -w
```

You should eventually see the Loki Pod running.

```text
NAME     READY   STATUS
loki-0   1/1     Running
```

Press:

```text
Ctrl + C
```

---

# 5. Verify the Loki Service

Run:

```bash
kubectl get svc -n loki
```

You should see a service named:

```text
loki
```

For example:

```text
NAME   TYPE        CLUSTER-IP      PORT(S)
loki   ClusterIP   10.x.x.x        3100/TCP
```

**This is the service Alloy will use.**

The Kubernetes DNS name is:

```text
loki.loki.svc.cluster.local
```

Therefore the Loki push endpoint is:

```text
http://loki.loki.svc.cluster.local:3100/loki/api/v1/push
```

---

# 6. Verify Loki Directly

Port-forward the Loki service:

```bash
kubectl port-forward -n loki svc/loki 3100:3100
```

Keep this terminal running.

Open another terminal:

```bash
curl http://localhost:3100/ready
```

Expected:

```text
ready
```

You can also check:

```bash
curl http://localhost:3100/metrics
```

If you receive Prometheus-style metrics, Loki is responding.

---

# Part 2 — Deploy Grafana Alloy

## 7. Add Grafana Helm repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Update:

```bash
helm repo update
```

---

# 8. Create Alloy namespace

```bash
kubectl create namespace alloy
```

---

# 9. Create Alloy configuration

Create:

```bash
vim alloy-values.yaml
```

Use:

```yaml
alloy:
  configMap:
    content: |
      logging {
        level  = "info"
        format = "logfmt"
      }

      discovery.kubernetes "pods" {
        role = "pod"
      }

      loki.source.kubernetes "pods" {
        targets = discovery.kubernetes.pods.targets

        forward_to = [
          loki.write.endpoint.receiver,
        ]
      }

      loki.write "endpoint" {
        endpoint {
          url = "http://loki.loki.svc.cluster.local:3100/loki/api/v1/push"
        }
      }
```

---

# 10. Understand the Alloy Pipeline

This is the most important part of the lab.

## Discovery

```alloy
discovery.kubernetes "pods" {
    role = "pod"
}
```

This discovers Kubernetes Pods.

The component continuously watches Kubernetes resources and produces targets. ([Grafana Labs][5])

---

## Source

```alloy
loki.source.kubernetes "pods" {
    targets = discovery.kubernetes.pods.targets

    forward_to = [
        loki.write.endpoint.receiver,
    ]
}
```

This collects logs from the discovered Pods.

`loki.source.kubernetes` tails Kubernetes container logs through the Kubernetes API. ([Grafana Labs][1])

Notice the connection:

```text
discovery.kubernetes.pods.targets
                  │
                  ▼
        loki.source.kubernetes
```

---

## Write

```alloy
loki.write "endpoint" {
    endpoint {
        url = "http://loki.loki.svc.cluster.local:3100/loki/api/v1/push"
    }
}
```

This sends the logs to Loki.

The connection is:

```text
loki.source.kubernetes
          │
          │ forward_to
          ▼
loki.write.endpoint.receiver
          │
          ▼
         Loki
```

---

# 11. Install Alloy

Run:

```bash
helm install alloy \
  grafana/alloy \
  -f alloy-values.yaml \
  -n alloy
```

Check:

```bash
kubectl get pods -n alloy
```

You should see:

```text
NAME                    READY   STATUS
alloy-xxxxxxxx          1/1     Running
```

---

# 12. Verify Alloy

Check Alloy logs:

```bash
kubectl logs -n alloy -l app.kubernetes.io/name=alloy
```

If the label doesn't match:

```bash
kubectl get pods -n alloy --show-labels
```

Then:

```bash
kubectl logs -n alloy <alloy-pod-name>
```

You should not see configuration or permission errors.

---

# Part 3 — Deploy an Application

Now we need an application that generates logs.

## 13. Create nginx Deployment

Create:

```bash
vim nginx.yaml
```

Add:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:latest

          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f nginx.yaml
```

Verify:

```bash
kubectl get pods
```

Expected:

```text
NAME                     READY   STATUS
nginx-xxxxxxxxxx-xxxxx   1/1     Running
nginx-xxxxxxxxxx-xxxxx   1/1     Running
```

---

# Part 4 — Generate Logs

## 14. Get an nginx Pod

```bash
kubectl get pods
```

Example:

```text
nginx-7d8b49557c-abcde
```

---

## 15. Generate HTTP requests

Run:

```bash
kubectl exec nginx-7d8b49557c-abcde -- curl localhost
```

Run it several times:

```bash
kubectl exec nginx-7d8b49557c-abcde -- curl localhost
```

```bash
kubectl exec nginx-7d8b49557c-abcde -- curl localhost
```

```bash
kubectl exec nginx-7d8b49557c-abcde -- curl localhost
```

---

# 16. Verify Kubernetes Logs

Run:

```bash
kubectl logs nginx-7d8b49557c-abcde
```

You should see nginx access logs similar to:

```text
10.244.0.10 - - [18/Aug/2026:04:30:20 +0000] "GET / HTTP/1.1" 200
```

This proves:

```text
nginx
  ↓
stdout
  ↓
Kubernetes container logs
```

Now Alloy should collect those logs.

---

# Part 5 — Verify Alloy → Loki

## 17. Check Alloy logs

```bash
kubectl logs -n alloy -l app.kubernetes.io/name=alloy
```

Look for errors.

If necessary:

```bash
kubectl describe pod -n alloy <alloy-pod>
```

---

# 18. Query Loki Directly

If your port-forward from earlier is still running:

```bash
curl "http://localhost:3100/loki/api/v1/labels"
```

If Loki is using multi-tenancy in your chart configuration, you may need the tenant header:

```bash
curl \
  -H "X-Scope-OrgID: local" \
  "http://localhost:3100/loki/api/v1/labels"
```

You should receive a response from Loki.

---

# Part 6 — Install Grafana

## 19. Add Grafana Helm repository

We already added:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Update:

```bash
helm repo update
```

---

# 20. Create Grafana namespace

```bash
kubectl create namespace monitoring
```

---

# 21. Install Grafana

```bash
helm install grafana \
  grafana/grafana \
  -n monitoring
```

Check:

```bash
kubectl get pods -n monitoring
```

Wait until:

```text
STATUS
Running
```

---

# 22. Get Grafana Password

Run:

```bash
kubectl get secret grafana \
  -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

Save the password.

Username:

```text
admin
```

---

# 23. Access Grafana

Port-forward:

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
```

Open:

```text
http://localhost:3000
```

Login:

```text
Username: admin
Password: <your-password>
```

---

# Part 7 — Configure Loki Data Source

In Grafana:

```text
Connections
      ↓
Data sources
      ↓
Add data source
      ↓
Loki
```

Use:

```text
URL:
http://loki.loki.svc.cluster.local:3100
```

Because Grafana is running inside Kubernetes, it can resolve the Loki Kubernetes Service using its DNS name.

Click:

```text
Save & Test
```

You should get:

```text
Data source connected successfully
```

---

# Part 8 — Query Logs

Go to:

```text
Explore
```

Select:

```text
Loki
```

Try:

```logql
{}
```

You should see Kubernetes logs.

---

## Query nginx logs

Try:

```logql
{container="nginx"}
```

Or:

```logql
{namespace="default"}
```

Or:

```logql
{namespace="default", container="nginx"}
```

You should see:

```text
GET /
```

requests generated earlier.

---

# Part 9 — Understand the Complete Pipeline

At this point, students should understand:

```text
                    Kubernetes
                         │
                         │
                    nginx Pod
                         │
                         │ stdout
                         ▼
                  Container Logs
                         │
                         ▼
             ┌──────────────────────┐
             │     Grafana Alloy    │
             │                      │
             │ discovery.kubernetes │
             │          ↓           │
             │ loki.source.kubernetes
             │          ↓           │
             │ loki.write            │
             └──────────┬───────────┘
                        │
                        │ HTTP
                        ▼
                 ┌─────────────┐
                 │    Loki     │
                 └──────┬──────┘
                        │
                        │ LogQL
                        ▼
                 ┌─────────────┐
                 │   Grafana   │
                 │   Explore   │
                 └─────────────┘
```

---

# Part 10 — Understand the Alloy Configuration

Students should be able to explain every connection:

```alloy
discovery.kubernetes "pods" {
    role = "pod"
}
```

### What does it do?

Discovers Kubernetes Pods.

---

```alloy
loki.source.kubernetes "pods" {
    targets = discovery.kubernetes.pods.targets
```

### What does this do?

Passes the discovered targets to the Kubernetes log source.

---

```alloy
forward_to = [
    loki.write.endpoint.receiver,
]
```

### What does this do?

Sends the collected logs to the receiver exposed by:

```text
loki.write "endpoint"
```

---

```alloy
loki.write "endpoint" {
```

### What does this do?

Creates the destination component.

---

```alloy
url = "http://loki.loki.svc.cluster.local:3100/loki/api/v1/push"
```

### What does this do?

Tells Alloy where to push the logs.

---

# Part 11 — Add `loki.process`

Once the basic pipeline works, **this is where I'd extend the lab**.

Change the pipeline to:

```text
discovery.kubernetes
        ↓
loki.source.kubernetes
        ↓
loki.process
        ↓
loki.write
        ↓
Loki
```

Add:

```alloy
loki.process "pods" {
    stage.static_labels {
        values = {
            app = "kubernetes"
        }
    }

    forward_to = [
        loki.write.endpoint.receiver,
    ]
}
```

Then change:

```alloy
loki.source.kubernetes "pods" {
    targets = discovery.kubernetes.pods.targets

    forward_to = [
        loki.process.pods.receiver,
    ]
}
```

Now the pipeline becomes:

```text
Source
  │
  ▼
loki.process
  │
  │ app="kubernetes"
  ▼
loki.write
  │
  ▼
Loki
```

Students can then query:

```logql
{app="kubernetes"}
```

This gives you a practical demonstration of the **Arguments → Receivers → Exports → Data Flow** concepts you've already taught.

---

# Part 12 — Troubleshooting

## Problem 1 — Loki Pod isn't ready

Check:

```bash
kubectl get pods -n loki
```

Then:

```bash
kubectl describe pod loki-0 -n loki
```

And:

```bash
kubectl logs -n loki loki-0
```

---

## Problem 2 — Loki returns 404

Check which service you're using:

```bash
kubectl get svc -n loki
```

For this lab, you should be using:

```text
loki
```

not:

```text
loki-gateway
```

We deliberately disabled:

```yaml
gateway:
  enabled: false
```

So the expected path is:

```text
Alloy
  ↓
loki.loki.svc.cluster.local:3100
  ↓
Loki
```

---

## Problem 3 — Alloy isn't collecting logs

Check:

```bash
kubectl logs -n alloy -l app.kubernetes.io/name=alloy
```

Then inspect Alloy's Pod:

```bash
kubectl describe pod -n alloy <alloy-pod>
```

Also verify that Alloy has Kubernetes RBAC permissions. The Alloy Helm chart can create the required `ClusterRole` and `ClusterRoleBinding` when RBAC is enabled. ([Grafana Labs][6])

Check:

```bash
kubectl get clusterrole | grep alloy
```

and:

```bash
kubectl get clusterrolebinding | grep alloy
```

---

## Problem 4 — No nginx logs

Check Kubernetes first:

```bash
kubectl logs <nginx-pod>
```

If Kubernetes itself doesn't show logs, the problem is **before Alloy**.

If Kubernetes shows logs but Grafana doesn't, investigate:

```text
Kubernetes
    ↓
Alloy
    ↓
Loki
    ↓
Grafana
```

---

# Final Architecture

The completed lab is:

```text
                         KUBERNETES
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  ┌───────────────────┐                                       │
│  │    nginx Pod      │                                       │
│  │                   │                                       │
│  │    nginx:latest   │                                       │
│  └─────────┬─────────┘                                       │
│            │                                                 │
│            │ stdout/stderr                                  │
│            ▼                                                 │
│  ┌─────────────────────────────┐                             │
│  │       Grafana Alloy         │                             │
│  │                             │                             │
│  │ discovery.kubernetes        │                             │
│  │             ↓               │                             │
│  │ loki.source.kubernetes      │                             │
│  │             ↓               │                             │
│  │ loki.process (optional)     │                             │
│  │             ↓               │                             │
│  │ loki.write                  │                             │
│  └──────────────┬──────────────┘                             │
│                 │                                            │
│                 │ HTTP POST /loki/api/v1/push               │
│                 ▼                                            │
│  ┌─────────────────────────────┐                            │
│  │            Loki             │                            │
│  │                             │                            │
│  │  Monolithic / single node   │                            │
│  │  Filesystem storage         │                            │
│  └──────────────┬──────────────┘                            │
│                 │                                            │
└─────────────────┼────────────────────────────────────────────┘
                  │
                  ▼
        ┌─────────────────────┐
        │       Grafana       │
        │                     │
        │      Explore        │
        │                     │
        │       LogQL         │
        └─────────────────────┘
```

## What students learn

By the end of this lab, students will have practiced:

```text
✓ Loki installation with Helm
✓ Loki monolithic deployment
✓ Kubernetes Services and DNS
✓ Grafana Alloy installation
✓ Alloy configuration
✓ discovery.kubernetes
✓ loki.source.kubernetes
✓ loki.process
✓ loki.write
✓ Arguments
✓ Component references
✓ Receivers
✓ forward_to
✓ Kubernetes RBAC
✓ Container log collection
✓ Loki ingestion
✓ LogQL
✓ Grafana Explore
✓ Troubleshooting an observability pipeline
```

**One deliberate change from your original README:** I would **not deploy Alloy as a DaemonSet for this particular `loki.source.kubernetes` lab**. That component can collect cluster-wide Pod logs through the Kubernetes API, so a single Alloy instance is much simpler for students and avoids multiple Alloy Pods independently watching the same cluster-wide targets. Grafana also notes that this component doesn't require a DaemonSet. ([Grafana Labs][1])

If you later teach the **node-local/DaemonSet architecture**, that's a separate excellent lab where you can introduce `loki.source.file`/`local.file_match` and `/var/log/pods`. Grafana's current Kubernetes logging guidance treats those as a separate collection pattern. ([Grafana Labs][7])

[1]: https://grafana.com/docs/grafana-cloud/send-data/alloy/reference/components/loki/loki.source.kubernetes/?utm_source=chatgpt.com "loki.source.kubernetes | Grafana Cloud documentation"
[2]: https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/?utm_source=chatgpt.com "Install the monolithic Helm chart | Grafana Loki documentation"
[3]: https://grafana.com/docs/loki/latest/setup/install/helm/?utm_source=chatgpt.com "Install Grafana Loki with Helm | Grafana Loki documentation"
[4]: https://grafana.com/docs/loki/latest/setup/install/helm/concepts/?utm_source=chatgpt.com "Helm chart components | Grafana Loki documentation"
[5]: https://grafana.com/docs/grafana-cloud/send-data/alloy/reference/components/discovery/discovery.kubernetes/?utm_source=chatgpt.com "discovery.kubernetes | Grafana Cloud documentation"
[6]: https://grafana.com/docs/alloy/latest/access_permissions/kubernetes/?utm_source=chatgpt.com "Access and permissions for Grafana Alloy on Kubernetes | Grafana Alloy documentation"
[7]: https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/?utm_source=chatgpt.com "Collect Kubernetes logs and forward them to Loki | Grafana Alloy documentation"
