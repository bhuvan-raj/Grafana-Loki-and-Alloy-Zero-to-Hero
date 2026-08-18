
# Loki + Grafana Alloy on Kubernetes

## 🎯 Lab Objective

In this lab, we will build a complete Kubernetes logging pipeline using:

* Kubernetes
* Grafana Alloy
* Grafana Loki
* Grafana

We will learn how to:

* Deploy Loki in monolithic mode
* Deploy Grafana Alloy
* Discover Kubernetes Pods using `discovery.kubernetes`
* Collect Pod logs using `loki.source.kubernetes`
* Process logs using `loki.process`
* Add labels to logs
* Forward logs using `forward_to`
* Send logs to Loki using `loki.write`
* Query logs using LogQL
* Visualize logs using Grafana Explore

---

# 🏗️ Lab Architecture

The final architecture will be:

```text
                         Kubernetes Cluster
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   ┌───────────────────┐                                      │
│   │   Application Pod │                                      │
│   │                   │                                      │
│   │   nginx           │                                      │
│   └─────────┬─────────┘                                      │
│             │                                                │
│             │ Container logs                                 │
│             ▼                                                │
│   ┌────────────────────────────────────────┐                 │
│   │             Grafana Alloy              │                 │
│   │                                        │                 │
│   │  discovery.kubernetes                  │                 │
│   │             ↓                          │                 │
│   │  loki.source.kubernetes                │                 │
│   │             ↓                          │                 │
│   │  loki.process                          │                 │
│   │             ↓                          │                 │
│   │  loki.write                             │                 │
│   └────────────────┬───────────────────────┘                 │
│                    │                                         │
│                    │ HTTP Push                               │
│                    ▼                                         │
│   ┌────────────────────────────────────────┐                 │
│   │               Grafana Loki             │                 │
│   │                                        │                 │
│   │          Log storage + querying        │                 │
│   └────────────────┬───────────────────────┘                 │
│                    │                                         │
└────────────────────┼─────────────────────────────────────────┘
                     │
                     │ LogQL
                     ▼
              ┌───────────────┐
              │    Grafana    │
              │    Explore    │
              └───────────────┘
```

## Alloy Pipeline

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

---

# 📋 Prerequisites

Students should have:

* Kubernetes cluster
* `kubectl`
* Helm 3
* Basic Kubernetes knowledge
* Basic Loki/LogQL knowledge

Verify Kubernetes:

```bash
kubectl cluster-info
```

Verify Helm:

```bash
helm version
```

---

# Part 1 — Deploy Loki

## 1. Create Loki Namespace

```bash
kubectl create namespace loki
```

Verify:

```bash
kubectl get namespace loki
```

Expected:

```text
NAME   STATUS
loki   Active
```

---

# 2. Add Grafana Community Helm Repository

Add the repository:

```bash
helm repo add grafana-community \
  https://grafana-community.github.io/helm-charts
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

# 3. Create Lab Directory

```bash
mkdir loki-alloy-lab
cd loki-alloy-lab
```

---

# 4. Create Loki Configuration

Create:

```bash
nano loki-values.yaml
```

Add:

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

# Disable gateway for this lab
gateway:
  enabled: false

chunksCache:
  enabled: false

resultsCache:
  enabled: false

test:
  enabled: false
```

### Why `deploymentMode: Monolithic`?

For this lab we want a simple single-node Loki deployment.

```text
Loki
└── Single Binary
```

This is suitable for learning and testing.

It is **not a production architecture**.

---

## Why `replication_factor: 1`?

We are running only one Loki replica:

```yaml
singleBinary:
  replicas: 1
```

Therefore:

```yaml
replication_factor: 1
```

is appropriate for this lab.

---

## Why disable the gateway?

We want the architecture to be:

```text
Alloy
   ↓
Loki Service
   ↓
Loki
```

instead of:

```text
Alloy
   ↓
Loki Gateway
   ↓
Loki
```

Therefore:

```yaml
gateway:
  enabled: false
```

This also makes the Kubernetes networking easier for students to understand.

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

Wait until Loki is running:

```bash
kubectl get pods -n loki -w
```

Expected:

```text
NAME     READY   STATUS
loki-0   2/2     Running
```

Press:

```text
Ctrl + C
```

---

# 6. Verify Loki Service

Run:

```bash
kubectl get svc -n loki
```

You should have:

```text
NAME   TYPE        CLUSTER-IP      PORT(S)
loki   ClusterIP   10.x.x.x        3100/TCP
```

The service DNS name is:

```text
loki.loki.svc.cluster.local
```

Therefore Loki's push endpoint is:

```text
http://loki.loki.svc.cluster.local:3100/loki/api/v1/push
```

---

# 7. Verify Loki from Inside Kubernetes

Instead of relying only on port-forwarding, test Loki from another Pod.

Run:

```bash
kubectl run curl-test \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- \
  curl -v \
  http://loki.loki.svc.cluster.local:3100/ready
```

Expected:

```text
HTTP/1.1 200 OK

ready
```

This verifies:

```text
Pod
 ↓
Kubernetes DNS
 ↓
Loki Service
 ↓
Loki
```

---

# 8. Understand Loki Multi-Tenancy

The Loki installation used in this lab requires a tenant ID.

Without the tenant header:

```bash
curl \
  http://loki.loki.svc.cluster.local:3100/loki/api/v1/labels
```

you may receive:

```text
401 Unauthorized

no org id
```

Therefore we use:

```text
X-Scope-OrgID: local
```

For example:

```bash
kubectl run curl-test \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- \
  curl \
  -H "X-Scope-OrgID: local" \
  http://loki.loki.svc.cluster.local:3100/loki/api/v1/labels
```

This should return:

```json
{
  "status": "success",
  "data": []
}
```

or a list of labels if logs have already been ingested.

---

# Part 2 — Deploy Grafana Alloy

# 9. Add Grafana Helm Repository

```bash
helm repo add grafana \
  https://grafana.github.io/helm-charts
```

Update:

```bash
helm repo update
```

---

# 10. Create Alloy Namespace

```bash
kubectl create namespace alloy
```

Verify:

```bash
kubectl get namespace alloy
```

---

# 11. Create Alloy Configuration

Create:

```bash
nano alloy-values.yaml
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

      // Discover Kubernetes Pods
      discovery.kubernetes "pods" {
        role = "pod"
      }

      // Collect logs from discovered Pods
      loki.source.kubernetes "pods" {
        targets = discovery.kubernetes.pods.targets

        forward_to = [
          loki.process.pods.receiver,
        ]
      }

      // Process and enrich logs
      loki.process "pods" {

        // Add a static label
        stage.static_labels {
          values = {
            app = "nginx"
          }
        }

        forward_to = [
          loki.write.endpoint.receiver,
        ]
      }

      // Send logs to Loki
      loki.write "endpoint" {
        endpoint {
          url       = "http://loki.loki.svc.cluster.local:3100/loki/api/v1/push"
          tenant_id = "local"
        }
      }
```

---

# 12. Understand the Alloy Configuration

This is the most important section of the lab.

## Component 1 — Kubernetes Discovery

```alloy
discovery.kubernetes "pods" {
  role = "pod"
}
```

This discovers Kubernetes Pods.

The component produces discovered targets.

The exported target data is accessed using:

```alloy
discovery.kubernetes.pods.targets
```

Conceptually:

```text
Kubernetes API
      ↓
discovery.kubernetes
      ↓
targets
```

---

# Component 2 — Kubernetes Log Source

```alloy
loki.source.kubernetes "pods" {
  targets = discovery.kubernetes.pods.targets

  forward_to = [
    loki.process.pods.receiver,
  ]
}
```

The source receives the targets:

```alloy
targets = discovery.kubernetes.pods.targets
```

and collects their container logs.

The important part is:

```alloy
forward_to = [
  loki.process.pods.receiver,
]
```

This sends the logs to the receiver exposed by:

```alloy
loki.process "pods"
```

---

# Component 3 — Loki Process

```alloy
loki.process "pods" {
```

This component is responsible for processing logs.

We are using:

```alloy
stage.static_labels
```

to add a label.

```alloy
stage.static_labels {
  values = {
    app = "nginx"
  }
}
```

Therefore logs passing through this component receive:

```text
app="nginx"
```

For example, conceptually:

Before processing:

```text
{namespace="default", pod="nginx-xxx"}
```

After processing:

```text
{namespace="default", pod="nginx-xxx", app="nginx"}
```

Then we forward the processed logs:

```alloy
forward_to = [
  loki.write.endpoint.receiver,
]
```

---

# Component 4 — Loki Write

```alloy
loki.write "endpoint" {
  endpoint {
    url       = "http://loki.loki.svc.cluster.local:3100/loki/api/v1/push"
    tenant_id = "local"
  }
}
```

This component sends the logs to Loki.

The endpoint is:

```text
http://loki.loki.svc.cluster.local:3100/loki/api/v1/push
```

The tenant is:

```text
local
```

---

# 13. Complete Alloy Data Flow

The configuration creates this pipeline:

```text
discovery.kubernetes
        │
        │ targets
        ▼
loki.source.kubernetes
        │
        │ logs
        ▼
loki.process
        │
        │ app="nginx"
        ▼
loki.write
        │
        │ HTTP Push
        ▼
       Loki
```

---

# 14. Install Alloy

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
NAME          READY   STATUS
alloy-xxxxx   2/2     Running
```

The Alloy Helm chart deploys Alloy as a DaemonSet by default.

Verify:

```bash
kubectl get daemonset -n alloy
```

Expected:

```text
NAME    DESIRED   CURRENT   READY
alloy   1         1         1
```

For a single-node lab cluster, one Alloy Pod is expected.

---

# 15. Verify Alloy Configuration

Check the ConfigMap:

```bash
kubectl get configmap alloy \
  -n alloy \
  -o yaml
```

You should see:

```text
discovery.kubernetes
loki.source.kubernetes
loki.process
loki.write
```

---

# 16. Check Alloy Logs

Run:

```bash
kubectl logs \
  -n alloy \
  -l app.kubernetes.io/name=alloy \
  --tail=50
```

You should see messages such as:

```text
tailer running
```

and:

```text
opened log stream
```

For example:

```text
component_id=loki.source.kubernetes.pods
target=default/nginx-xxxxx:nginx
```

This confirms Alloy is discovering and opening Kubernetes log streams.

---

# Part 3 — Deploy nginx

# 17. Create nginx Deployment

Create:

```bash
nano nginx.yaml
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

Check:

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

# 18. Generate nginx Logs

Get the Pods:

```bash
kubectl get pods
```

Example:

```text
nginx-59f86b59ff-5rrzx
nginx-59f86b59ff-v5rsd
```

Generate requests:

```bash
kubectl exec nginx-59f86b59ff-5rrzx -- curl localhost
```

Run multiple requests:

```bash
kubectl exec nginx-59f86b59ff-5rrzx -- curl localhost
```

```bash
kubectl exec nginx-59f86b59ff-5rrzx -- curl localhost
```

```bash
kubectl exec nginx-59f86b59ff-v5rsd -- curl localhost
```

---

# 19. Verify Kubernetes Logs

Check:

```bash
kubectl logs nginx-59f86b59ff-5rrzx
```

You should see nginx access logs:

```text
10.244.x.x - - [18/Aug/2026:09:00:00 +0000] "GET / HTTP/1.1" 200 ...
```

This proves:

```text
nginx
 ↓
stdout
 ↓
Kubernetes container logs
```

Now Alloy should collect these logs.

---

# Part 4 — Verify Alloy → Loki

# 20. Check Alloy Log Collection

Run:

```bash
kubectl logs \
  -n alloy \
  -l app.kubernetes.io/name=alloy \
  --tail=50
```

Look for:

```text
target=default/nginx-xxxxx:nginx
```

and:

```text
opened log stream
```

This confirms:

```text
Kubernetes
    ↓
Alloy
```

---

# 21. Check Loki Labels

Run:

```bash
kubectl run curl-test \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- \
  curl \
  -H "X-Scope-OrgID: local" \
  http://loki.loki.svc.cluster.local:3100/loki/api/v1/labels
```

Expected output should include:

```json
{
  "status": "success",
  "data": [
    "app",
    "instance",
    "job",
    "service_name"
  ]
}
```

The important part is:

```text
app
```

This proves that our `loki.process` stage added the label.

---

# Part 5 — Query Logs Directly from Loki

# 22. Query Logs for the nginx Application

Loki provides the `/loki/api/v1/query_range` API.

We can use it to query logs directly.

For example:

```bash
kubectl run curl-test \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- \
  curl \
  -G \
  -H "X-Scope-OrgID: local" \
  --data-urlencode 'query={app="nginx"}' \
  http://loki.loki.svc.cluster.local:3100/loki/api/v1/query_range
```

If logs have been ingested, Loki will return the matching streams.

---

# 23. Query Logs for a Specific Pod

First:

```bash
kubectl get pods
```

Suppose the Pod is:

```text
nginx-59f86b59ff-5rrzx
```

Use LogQL:

```logql
{app="nginx", pod="nginx-59f86b59ff-5rrzx"}
```

You can test this in Grafana later.

---

# Part 6 — Install Grafana

# 24. Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

---

# 25. Install Grafana

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

# 26. Get Grafana Password

Run:

```bash
kubectl get secret grafana \
  -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

Username:

```text
admin
```

---

# 27. Access Grafana

Port-forward:

```bash
kubectl port-forward \
  -n monitoring \
  svc/grafana \
  3000:80
```

Open:

```text
http://localhost:3000
```

Login:

```text
Username: admin
Password: <password>
```

---

# Part 7 — Configure Loki in Grafana

# 28. Add Loki Data Source

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

Set:

```text
URL:

http://loki.loki.svc.cluster.local:3100
```

Because Grafana is running inside Kubernetes, it can resolve:

```text
loki.loki.svc.cluster.local
```

through Kubernetes DNS.

---

# 29. Configure Loki Tenant

Because our Loki installation requires a tenant ID, configure the HTTP header.

Add:

```text
Header:

X-Scope-OrgID
```

Value:

```text
local
```

So Grafana sends:

```http
X-Scope-OrgID: local
```

to Loki.

Click:

```text
Save & Test
```

You should get:

```text
Data source connected successfully
```

---

# Part 8 — Query Logs Using Grafana

# 30. Open Explore

Go to:

```text
Explore
```

Select:

```text
Loki
```

---

# 31. Query All nginx Logs

Use:

```logql
{app="nginx"}
```

You should see nginx logs.

---

# 32. Query a Specific Pod

Get the Pod name:

```bash
kubectl get pods
```

For example:

```text
nginx-59f86b59ff-5rrzx
```

Use:

```logql
{app="nginx", pod="nginx-59f86b59ff-5rrzx"}
```

This returns logs only from that specific Pod.

---

# 33. Query Both nginx Pods

You can use a regular expression:

```logql
{app="nginx", pod=~"nginx-.*"}
```

This matches all Pods whose name begins with:

```text
nginx-
```

---

# 34. Search for HTTP Requests

You can filter log content:

```logql
{app="nginx"} |= "GET"
```

Or:

```logql
{app="nginx"} |= "200"
```

For example:

```logql
{app="nginx"} |= "GET /"
```

---

# Part 9 — Understand `loki.process`

Now students should understand why the process component exists.

Our current pipeline is:

```text
SOURCE
   ↓
PROCESS
   ↓
WRITE
```

Specifically:

```text
loki.source.kubernetes
        ↓
loki.process
        ↓
loki.write
```

---

# 35. Static Labels

Our process configuration is:

```alloy
loki.process "pods" {

  stage.static_labels {
    values = {
      app = "nginx"
    }
  }

  forward_to = [
    loki.write.endpoint.receiver,
  ]
}
```

The stage:

```alloy
stage.static_labels
```

adds:

```text
app="nginx"
```

to the logs.

---

# 36. Why Process Logs?

`loki.process` can be used for many types of log processing.

For example:

```text
Parse logs
     ↓
Extract fields
     ↓
Add labels
     ↓
Drop unwanted logs
     ↓
Rewrite labels
     ↓
Add structured metadata
     ↓
Forward to Loki
```

Some commonly used processing stages include:

```text
stage.static_labels
stage.json
stage.regex
stage.logfmt
stage.labels
stage.drop
stage.timestamp
stage.replace
stage.match
```

This gives students a foundation for more advanced Alloy pipelines.

---

# Part 10 — Understanding Alloy Components

Students should understand the naming convention.

For example:

```alloy
loki.process "pods"
```

has:

```text
Component type:
loki.process

Component label:
pods
```

The full component reference is:

```text
loki.process.pods
```

Its receiver is:

```text
loki.process.pods.receiver
```

Similarly:

```alloy
loki.write "endpoint"
```

becomes:

```text
loki.write.endpoint
```

and exposes:

```text
loki.write.endpoint.receiver
```

---

# 37. Understanding `forward_to`

This:

```alloy
forward_to = [
  loki.process.pods.receiver,
]
```

means:

> Send the data to the receiver of `loki.process.pods`.

Then:

```alloy
loki.process "pods" {
    ...
    
    forward_to = [
      loki.write.endpoint.receiver,
    ]
}
```

means:

> Send the processed data to the receiver of `loki.write.endpoint`.

Therefore:

```text
loki.source
     │
     │ forward_to
     ▼
loki.process
     │
     │ forward_to
     ▼
loki.write
```

---

# 38. Arguments, Receivers and Exports

This lab also demonstrates three important Alloy concepts.

## Arguments

For example:

```alloy
targets = discovery.kubernetes.pods.targets
```

`targets` is an argument of:

```text
loki.source.kubernetes
```

Another example:

```alloy
url = "http://loki.loki.svc.cluster.local:3100/loki/api/v1/push"
```

`url` is an argument of the Loki endpoint.

---

## Receivers

This:

```alloy
loki.process.pods.receiver
```

is a receiver.

It is where another component sends data.

For example:

```alloy
forward_to = [
  loki.process.pods.receiver,
]
```

---

## Exports

A component can expose data for other components.

For example:

```alloy
discovery.kubernetes.pods.targets
```

is an exported value from:

```alloy
discovery.kubernetes "pods"
```

which is consumed by:

```alloy
loki.source.kubernetes
```

Therefore:

```text
discovery.kubernetes.pods.targets
```

is an example of a component export.

---

# Part 11 — Troubleshooting

## Problem 1 — Loki installation fails with replica error

If you see:

```text
You have more than zero replicas configured for both
the monolithic and simple scalable targets.
```

make sure your values file disables the other deployment modes:

```yaml
backend:
  replicas: 0

read:
  replicas: 0

write:
  replicas: 0
```

and:

```yaml
singleBinary:
  replicas: 1
```

Also ensure:

```yaml
deploymentMode: Monolithic
```

---

# Problem 2 — Loki returns `401 no org id`

If:

```bash
curl http://loki.loki.svc.cluster.local:3100/loki/api/v1/labels
```

returns:

```text
401 Unauthorized
no org id
```

add:

```http
X-Scope-OrgID: local
```

Example:

```bash
curl \
  -H "X-Scope-OrgID: local" \
  http://loki.loki.svc.cluster.local:3100/loki/api/v1/labels
```

And make sure Alloy contains:

```yaml
tenant_id: "local"
```

inside the Loki endpoint.

---

# Problem 3 — Alloy reports `no org id`

If Alloy logs contain:

```text
status=401
error="server returned HTTP status 401 Unauthorized (401): no org id"
```

check:

```alloy
loki.write "endpoint" {
  endpoint {
    url       = "http://loki.loki.svc.cluster.local:3100/loki/api/v1/push"
    tenant_id = "local"
  }
}
```

Then upgrade Alloy:

```bash
helm upgrade alloy \
  grafana/alloy \
  -f alloy-values.yaml \
  -n alloy
```

Check:

```bash
kubectl get configmap alloy -n alloy -o yaml
```

Verify:

```text
tenant_id = "local"
```

---

# Problem 4 — Alloy isn't collecting logs

Check:

```bash
kubectl logs \
  -n alloy \
  -l app.kubernetes.io/name=alloy \
  --tail=50
```

Look for:

```text
tailer running
```

and:

```text
opened log stream
```

For example:

```text
target=default/nginx-xxxxx:nginx
```

---

# Problem 5 — Alloy has RBAC errors

Check:

```bash
kubectl get clusterrole | grep alloy
```

and:

```bash
kubectl get clusterrolebinding | grep alloy
```

Also inspect:

```bash
kubectl describe pod -n alloy <alloy-pod>
```

---

# Problem 6 — Loki Service isn't reachable

Check:

```bash
kubectl get svc -n loki
```

You should see:

```text
loki
```

Check endpoints:

```bash
kubectl get endpoints -n loki
```

You should see something similar to:

```text
loki   10.244.x.x:3100
```

Then test:

```bash
kubectl run curl-test \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- \
  curl \
  http://loki.loki.svc.cluster.local:3100/ready
```

Expected:

```text
ready
```

---

# Problem 7 — Grafana cannot connect to Loki

Verify the Loki service:

```bash
kubectl get svc -n loki
```

Use:

```text
http://loki.loki.svc.cluster.local:3100
```

Do **not** use:

```text
http://loki-gateway.loki.svc.cluster.local
```

because this lab disables the gateway.

Also configure:

```text
X-Scope-OrgID: local
```

in the Grafana Loki data source.

---

# Problem 8 — Grafana shows no logs

Troubleshoot from left to right:

```text
Kubernetes
    ↓
Alloy
    ↓
Loki
    ↓
Grafana
```

First check:

```bash
kubectl logs <nginx-pod>
```

If that works, check Alloy:

```bash
kubectl logs -n alloy -l app.kubernetes.io/name=alloy
```

Then query Loki:

```bash
kubectl run curl-test \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- \
  curl \
  -H "X-Scope-OrgID: local" \
  http://loki.loki.svc.cluster.local:3100/loki/api/v1/labels
```

Finally check Grafana.

---

# 🧹 Cleanup

When the lab is complete:

Delete nginx:

```bash
kubectl delete -f nginx.yaml
```

Delete Grafana:

```bash
helm uninstall grafana -n monitoring
```

Delete Alloy:

```bash
helm uninstall alloy -n alloy
```

Delete Loki:

```bash
helm uninstall loki -n loki
```

Delete namespaces:

```bash
kubectl delete namespace monitoring
kubectl delete namespace alloy
kubectl delete namespace loki
```

---

# 🎓 What Students Learn

By completing this lab, students will understand:

```text
✓ What Grafana Alloy is
✓ Alloy component architecture
✓ Alloy configuration syntax
✓ Component naming
✓ Arguments
✓ Exports
✓ Receivers
✓ forward_to
✓ Kubernetes discovery
✓ Kubernetes log collection
✓ loki.source.kubernetes
✓ loki.process
✓ Processing stages
✓ Static labels
✓ loki.write
✓ Loki ingestion
✓ Loki tenants
✓ Kubernetes Service DNS
✓ Kubernetes RBAC
✓ LogQL
✓ Grafana Explore
✓ End-to-end troubleshooting
```

## Final Concept

The most important thing students should take away is:

```text
                ALLOY

     Discover
        ↓
  discovery.kubernetes
        ↓
     Collect
        ↓
loki.source.kubernetes
        ↓
     Process
        ↓
   loki.process
        ↓
      Write
        ↓
    loki.write
        ↓
       LOKI
        ↓
     Query
        ↓
     GRAFANA
```

This gives you a **proper end-to-end Alloy lab** rather than just installing Alloy and forwarding logs.
