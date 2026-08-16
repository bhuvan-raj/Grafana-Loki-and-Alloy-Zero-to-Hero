
#Lab Architecture

```text
                  Kubernetes Cluster
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   ┌─────────────────┐                                    │
│   │ Application Pod │                                    │
│   │                 │                                    │
│   │ nginx / app     │                                    │
│   └────────┬────────┘                                    │
│            │                                             │
│            │ Container logs                              │
│            ▼                                             │
│   ┌──────────────────────┐                               │
│   │    Grafana Alloy     │                               │
│   │                      │                               │
│   │ discovery.kubernetes │                               │
│   │          ↓           │                               │
│   │ loki.source          │                               │
│   │ .kubernetes          │                              │
│   │          ↓           │                               │
│   │ loki.write           │                               │
│   └──────────┬───────────┘                               │
│              │                                           │
│              │ HTTP Push                                 │
│              ▼                                           │
│   ┌──────────────────────┐                               │
│   │     Grafana Loki     │                               │
│   │                      │                               │
│   │ Log storage/query    │                               │
│   └──────────┬───────────┘                               │
│              │                                           │
└──────────────┼───────────────────────────────────────────┘
               │
               ▼
        ┌──────────────┐
        │   Grafana    │
        │    Explore   │
        └──────────────┘
```

---

# Lab — Deploy Loki and Grafana Alloy on Kubernetes

## 1. Prerequisites

Students should have:

* Kubernetes cluster
* `kubectl`
* Helm 3
* Basic Kubernetes knowledge


---

# Part 1 — Deploy Loki

## 2. Create Loki namespace

```bash
kubectl create namespace loki
```

Verify:

```bash
kubectl get namespaces
```

---

## 3. Add the Grafana Community Helm repository

The Loki Helm chart is currently maintained in the Grafana Community Helm repository. ([Grafana Labs][1])

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
```

Update:

```bash
helm repo update
```

Verify:

```bash
helm search repo grafana-community/loki
```

---

# 4. Create Loki configuration

Create:

```bash
mkdir loki-alloy-lab
cd loki-alloy-lab

vim loki-values.yaml
```

Put:

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

gateway:
  enabled: true

chunksCache:
  enabled: false

resultsCache:
  enabled: false

test:
  enabled: false
```

### Why `replication_factor: 1`?

This is a **single-replica lab**.

Grafana's current Loki documentation specifically notes that a single-replica monolithic deployment should use:

```yaml
commonConfig:
  replication_factor: 1
```

Otherwise Loki requests can fail. ([Grafana Labs][1])

This setup is for **learning/testing**, not production.

---

# 5. Install Loki

Run:

```bash
helm install loki \
  grafana-community/loki \
  -f loki-values.yaml \
  -n loki
```

Check:

```bash
kubectl get pods -n loki
```

You should eventually see something similar to:

```text
NAME                            READY   STATUS
loki-0                          1/1     Running
loki-gateway-xxxxx              1/1     Running
```

Also:

```bash
kubectl get svc -n loki
```

You should see services including the Loki gateway.

---

# 6. Verify Loki

Check the Loki gateway:

```bash
kubectl get svc -n loki
```

Find the gateway service:

```bash
kubectl get svc -n loki | grep gateway
```

You can test Loki internally using port forwarding.

For example:

```bash
kubectl port-forward -n loki svc/loki-gateway 3100:80
```

Then from another terminal:

```bash
curl http://localhost:3100/ready
```

Expected:

```text
ready
```

---

# Part 2 — Deploy Grafana Alloy

Now students install Alloy.

Grafana provides an official Helm chart for deploying Alloy into Kubernetes. ([Grafana Labs][3])

## 7. Add Grafana Helm repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Then:

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
  mounts:
    varlog: true

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
          url = "http://loki-gateway.loki.svc.cluster.local/loki/api/v1/push"
          tenant_id = "local"
        }
      }
```

This is the key part of the lab.

The pipeline is:

```text
discovery.kubernetes
        ↓
loki.source.kubernetes
        ↓
loki.write
        ↓
Loki
```

Grafana's current Loki documentation provides essentially this same architecture for collecting Kubernetes Pod logs with Alloy. ([Grafana Labs][4])

---

# 10. Understand `discovery.kubernetes`

This:

```alloy
discovery.kubernetes "pods" {
  role = "pod"
}
```

tells Alloy:

> Discover Kubernetes Pods.

Alloy gets metadata such as:

```text
namespace
pod name
container name
pod UID
labels
```

The `discovery.kubernetes` component provides targets that can then be consumed by `loki.source.kubernetes`. ([Grafana Labs][5])

---

# 11. Understand `loki.source.kubernetes`

This:

```alloy
loki.source.kubernetes "pods" {
  targets = discovery.kubernetes.pods.targets

  forward_to = [
    loki.write.endpoint.receiver,
  ]
}
```

means:

> Take the discovered Kubernetes Pods and collect their container logs.

`loki.source.kubernetes` uses the Kubernetes API to tail container logs. It does not require Alloy to have direct access to the node's log filesystem. ([Grafana Labs][6])

That's a very useful concept to teach.

---

# 12. Understand `loki.write`

Finally:

```alloy
loki.write "endpoint" {
  endpoint {
    url = "http://loki-gateway.loki.svc.cluster.local/loki/api/v1/push"
    tenant_id = "local"
  }
}
```

This tells Alloy:

> Send the collected logs to Loki.

So:

```text
loki.source.kubernetes
              │
              │ logs
              ▼
        loki.write
              │
              │ HTTP POST
              ▼
      Loki /push endpoint
```

---

# 13. Install Alloy

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

Expected:

```text
NAME                     READY   STATUS
alloy-xxxxxxxx           1/1     Running
```

Check:

```bash
kubectl get daemonset -n alloy
```

Depending on the Helm chart configuration/version, the deployment mode may differ, so students should inspect what Helm actually created rather than assuming a particular controller.

---

# Part 3 — Create an Application

Now we need something that generates logs.

## 14. Deploy nginx

Create:

```bash
vim nginx.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
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

---

# 15. Generate some logs

Get a Pod:

```bash
kubectl get pods
```

Then generate requests:

```bash
kubectl exec -it <nginx-pod> -- curl localhost
```

Run it multiple times:

```bash
kubectl exec -it <nginx-pod> -- curl localhost
kubectl exec -it <nginx-pod> -- curl localhost
kubectl exec -it <nginx-pod> -- curl localhost
```

Check the native Kubernetes logs:

```bash
kubectl logs <nginx-pod>
```

Students should see something similar to:

```text
10.244.0.10 - - [16/Aug/2026:07:20:12 +0000] "GET / HTTP/1.1" 200 ...
```

---

# Part 4 — Verify Alloy

Check Alloy logs:

```bash
kubectl logs -n alloy -l app.kubernetes.io/name=alloy
```

If the label selector differs in the installed chart, first run:

```bash
kubectl get pods -n alloy --show-labels
```

Then use the appropriate Pod name:

```bash
kubectl logs -n alloy <alloy-pod>
```

You want to make sure Alloy is running without configuration errors.

---

# Part 5 — Verify Loki Has Logs

Port-forward the Loki gateway:

```bash
kubectl port-forward -n loki svc/loki-gateway 3100:80
```

Then:

```bash
curl http://localhost:3100/loki/api/v1/labels \
  -H "X-Scope-OrgID: local"
```

You should receive a JSON response containing labels.

For example:

```json
{
  "status": "success",
  "data": [
    "container",
    "namespace",
    "pod"
  ]
}
```

The exact labels will depend on your cluster and Alloy configuration.

---

# Part 6 — Add Grafana

For the lab, you can also deploy Grafana.

```bash
helm install grafana grafana/grafana \
  -n monitoring \
  --create-namespace
```

Check:

```bash
kubectl get pods -n monitoring
```

Get the Grafana password:

```bash
kubectl get secret grafana \
  -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

Port-forward:

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
```

Then open:

```text
http://localhost:3000
```

Login:

```text
Username: admin
Password: <password-from-secret>
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
http://loki-gateway.loki.svc.cluster.local
```

Because Grafana is inside Kubernetes, it can communicate with Loki using the Kubernetes Service DNS name.

For this Loki setup, configure the tenant header:

```text
Header:
X-Scope-OrgID

Value:
local
```

Grafana's documentation notes that Loki deployments using Helm on Kubernetes have multi-tenancy enabled by default, so the tenant ID needs to be supplied when querying. ([Grafana Labs][7])

Click:

```text
Save & Test
```

---

# Part 8 — View Logs

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
{namespace="default"}
```

Or:

```logql
{container="nginx"}
```

You should see the nginx logs.

You can also try:

```logql
{pod=~"nginx-.*"}
```

And:

```logql
{namespace="default", container="nginx"}
```

---

# What Students Should Understand

At the end of the lab, students should be able to explain this:

```text
Application
     │
     │ writes stdout/stderr
     ▼
Kubernetes
     │
     │ Pod logs
     ▼
Grafana Alloy
     │
     ├── discovery.kubernetes
     │
     ├── loki.source.kubernetes
     │
     └── loki.write
     │
     ▼
Grafana Loki
     │
     │ LogQL
     ▼
Grafana
     │
     ▼
Explore
```

## Important distinction

I'd specifically emphasize this to your students:

| Component         | Responsibility                                      |
| ----------------- | --------------------------------------------------- |
| Kubernetes        | Runs applications and produces container logs       |
| **Grafana Alloy** | Collects, processes and forwards logs               |
| **Grafana Loki**  | Stores and indexes log metadata / provides querying |
| **LogQL**         | Queries Loki                                        |
| **Grafana**       | Visualizes and explores logs                        |

Grafana's documentation describes Alloy as the recommended agent for sending logs to Loki, with components such as `discovery.kubernetes`, `loki.source.kubernetes`, and `loki.write` forming the collection pipeline. ([Grafana Labs][4])

## Official References

[1]: https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/?utm_source=chatgpt.com "Install the monolithic Helm chart | Grafana Loki documentation"
[2]: https://grafana.com/docs/loki/latest/setup/install/local/?utm_source=chatgpt.com "Install Grafana Loki locally | Grafana Loki documentation"
[3]: https://grafana.com/docs/alloy/latest/set-up/install/kubernetes/?plcmt=products-nav&utm_source=chatgpt.com "Deploy Grafana Alloy on Kubernetes | Grafana Alloy documentation"
[4]: https://grafana.com/docs/loki/latest/get-started/?utm_source=chatgpt.com "Get started with Grafana Loki | Grafana Loki documentation"
[5]: https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/?utm_source=chatgpt.com "loki.source.kubernetes | Grafana Alloy documentation"
[6]: https://grafana.com/docs/grafana-cloud/send-data/alloy/reference/components/loki/loki.source.kubernetes/?utm_source=chatgpt.com "loki.source.kubernetes | Grafana Cloud documentation"
[7]: https://grafana.com/docs/loki/latest/visualize/grafana/?utm_source=chatgpt.com "Visualize log data | Grafana Loki documentation"
