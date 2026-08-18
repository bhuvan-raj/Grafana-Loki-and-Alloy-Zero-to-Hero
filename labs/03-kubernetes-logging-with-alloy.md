
# Loki + Grafana Alloy on Kubernetes

## 🎯 Lab Objective

In this lab, we will build a complete Kubernetes logging pipeline using:

- Kubernetes
- Grafana Alloy
- Grafana Loki
- Grafana
- LogQL

By the end of this lab, students will understand how Grafana Alloy discovers Kubernetes Pods, collects their logs, optionally processes them, and forwards them to Loki.

---

# 🏗️ Final Architecture

```text
                         Kubernetes Cluster
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   ┌───────────────────┐                                     │
│   │  Application Pod  │                                     │
│   │                   │                                     │
│   │      nginx        │                                     │
│   └─────────┬─────────┘                                     │
│             │                                               │
│             │ Container logs                                │
│             ▼                                               │
│   ┌──────────────────────────────┐                          │
│   │        Grafana Alloy         │                          │
│   │                              │                          │
│   │  discovery.kubernetes        │                          │
│   │             ↓                │                          │
│   │  loki.source.kubernetes      │                          │
│   │             ↓                │                          │
│   │  loki.process (optional)     │                          │
│   │             ↓                │                          │
│   │  loki.write                  │                          │
│   └──────────────┬───────────────┘                          │
│                  │                                          │
│                  │ HTTP Push                               │
│                  ▼                                          │
│   ┌──────────────────────────────┐                          │
│   │          Grafana Loki        │                          │
│   │                              │                          │
│   │     Log storage + query      │                          │
│   └──────────────┬───────────────┘                          │
│                  │                                          │
└──────────────────┼──────────────────────────────────────────┘
                   │
                   │ LogQL
                   ▼
          ┌──────────────────┐
          │     Grafana      │
          │     Explore      │
          └──────────────────┘
````

---

# 🔄 Alloy Pipeline

The basic pipeline is:

```text
discovery.kubernetes
        ↓
loki.source.kubernetes
        ↓
loki.write
        ↓
       Loki
```

After introducing processing:

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

# 📚 What You Will Learn

By completing this lab, you will learn:

* Installing Loki using Helm
* Deploying Loki in monolithic mode
* Kubernetes Services
* Kubernetes DNS
* Installing Grafana Alloy
* Alloy configuration syntax
* Alloy components
* Arguments
* Receivers
* Exports
* `forward_to`
* `discovery.kubernetes`
* `loki.source.kubernetes`
* `loki.process`
* `loki.write`
* Kubernetes log collection
* Loki multi-tenancy
* `X-Scope-OrgID`
* LogQL
* Grafana Explore
* Troubleshooting the complete logging pipeline

---

# Prerequisites

Students should have:

* Kubernetes cluster
* `kubectl`
* Helm 3
* Basic Kubernetes knowledge
* Basic Loki knowledge
* Basic LogQL knowledge

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

# 2. Add the Grafana Community Helm Repository

Add the Loki Helm repository:

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts
```

Update repositories:

```bash
helm repo update
```

Verify:

```bash
helm search repo grafana-community/loki
```

---

# 3. Create Loki Values File

Create a working directory:

```bash
mkdir loki-alloy-lab
cd loki-alloy-lab
```

Create:

```bash
nano loki-values.yaml
```

Use the following configuration:

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

---

# 4. Understanding the Loki Configuration

## Deployment Mode

```yaml
deploymentMode: Monolithic
```

For this lab, Loki runs as a single process.

Architecture:

```text
┌─────────────────┐
│      Loki       │
│                 │
│ Distributor     │
│ Ingester        │
│ Querier         │
│ Query Frontend  │
│ Compactor       │
│ etc.            │
└─────────────────┘
```

This keeps the lab simple.

It is suitable for learning and testing, not production-scale deployments.

---

## Replication Factor

```yaml
replication_factor: 1
```

We have only one Loki replica:

```text
Loki
 └── 1 replica
```

Therefore:

```yaml
replication_factor: 1
```

is appropriate for this lab.

---

## Filesystem Storage

```yaml
storage:
  type: filesystem
```

Loki stores data locally inside the Loki container's filesystem.

This is suitable for this lab.

For production deployments, object storage such as S3, GCS, or Azure Blob Storage is commonly used.

---

## Disable Gateway

```yaml
gateway:
  enabled: false
```

We deliberately disable the Loki gateway.

Therefore our architecture is:

```text
Alloy
  ↓
Kubernetes Service
  ↓
Loki
```

Instead of:

```text
Alloy
  ↓
Loki Gateway
  ↓
Loki
```

The Loki Service will be:

```text
loki.loki.svc.cluster.local
```

---

# 5. Install Loki

Run:

```bash
helm install loki \
  grafana-community/loki \
  -f loki-values.yaml \
  -n loki
```

Check the Pods:

```bash
kubectl get pods -n loki
```

Expected:

```text
NAME     READY   STATUS
loki-0   2/2     Running
```

Depending on the chart version, Loki may contain more than one container.

Wait until the Pod is fully ready.

---

# 6. Verify Loki Service

Run:

```bash
kubectl get svc -n loki
```

You should see:

```text
NAME   TYPE        CLUSTER-IP      PORT(S)
loki   ClusterIP   10.x.x.x        3100/TCP
```

The important Service is:

```text
loki
```

Its fully qualified Kubernetes DNS name is:

```text
loki.loki.svc.cluster.local
```

Therefore:

```text
Loki URL:

http://loki.loki.svc.cluster.local:3100
```

---

# 7. Verify Loki Endpoints

Run:

```bash
kubectl get endpoints -n loki
```

You should see an endpoint for:

```text
loki
```

For newer Kubernetes versions, you can also inspect EndpointSlices:

```bash
kubectl get endpointslice -n loki
```

---

# 8. Test Loki from Inside the Cluster

This is better than relying only on port-forwarding because Alloy will access Loki from inside Kubernetes.

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

This proves:

```text
Pod
 ↓
Kubernetes DNS
 ↓
Loki Service
 ↓
Loki
```

is working.

---

# 9. Test Loki API

Loki may require a tenant ID.

Test:

```bash
kubectl run curl-test \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- \
  curl -v \
  http://loki.loki.svc.cluster.local:3100/loki/api/v1/labels
```

If you receive:

```text
401 Unauthorized
no org id
```

this is expected for a multi-tenant Loki configuration.

Test again with the tenant:

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

Expected:

```json
{
  "status": "success",
  "data": []
}
```

An empty data array is fine at this stage because we haven't sent logs yet.

---

# Part 2 — Deploy Grafana Alloy

# 10. Add Grafana Helm Repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Update:

```bash
helm repo update
```

Verify:

```bash
helm search repo grafana/alloy
```

---

# 11. Create Alloy Namespace

```bash
kubectl create namespace alloy
```

Verify:

```bash
kubectl get namespace alloy
```

---

# 12. Create Alloy Configuration

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
          url       = "http://loki.loki.svc.cluster.local:3100/loki/api/v1/push"
          tenant_id = "local"
        }
      }
```

---

# 13. Understand the Alloy Configuration

## Logging Component

```alloy
logging {
  level  = "info"
  format = "logfmt"
}
```

Controls Alloy's own logs.

It does not process Kubernetes application logs.

---

# 14. Kubernetes Discovery

```alloy
discovery.kubernetes "pods" {
  role = "pod"
}
```

This discovers Kubernetes Pods.

The component produces discovered targets.

The component name is:

```text
discovery.kubernetes.pods
```

It can be referenced using:

```alloy
discovery.kubernetes.pods.targets
```

---

# 15. Kubernetes Log Source

```alloy
loki.source.kubernetes "pods" {
  targets = discovery.kubernetes.pods.targets

  forward_to = [
    loki.write.endpoint.receiver,
  ]
}
```

This component receives the discovered Pod targets and collects their Kubernetes container logs.

The data flow is:

```text
discovery.kubernetes.pods.targets
              ↓
loki.source.kubernetes
```

---

# 16. Forward To

```alloy
forward_to = [
  loki.write.endpoint.receiver,
]
```

`forward_to` specifies where the logs should be sent.

Here:

```text
loki.write.endpoint.receiver
```

is the receiver exported by:

```alloy
loki.write "endpoint"
```

So:

```text
loki.source.kubernetes
          ↓
loki.write.endpoint.receiver
```

---

# 17. Loki Write Component

```alloy
loki.write "endpoint" {
  endpoint {
    url       = "http://loki.loki.svc.cluster.local:3100/loki/api/v1/push"
    tenant_id = "local"
  }
}
```

This defines where Alloy sends the logs.

The URL:

```text
http://loki.loki.svc.cluster.local:3100/loki/api/v1/push
```

is Loki's Push API.

---

# 18. Tenant ID

```alloy
tenant_id = "local"
```

Our Loki installation requires a tenant ID.

Alloy uses:

```text
local
```

as the tenant.

Conceptually, Alloy sends:

```text
X-Scope-OrgID: local
```

to Loki.

Without this, Loki responds:

```text
401 Unauthorized
no org id
```

---

# 19. Install Alloy

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

The exact number of containers depends on the Alloy Helm chart version.

---

# 20. Check Alloy DaemonSet

The Helm chart may deploy Alloy as a DaemonSet depending on its configuration.

Check:

```bash
kubectl get deployment,statefulset,daemonset -n alloy
```

For the lab configuration used here, you should verify the actual controller created by Helm instead of assuming it.

If it is a DaemonSet:

```text
DaemonSet
   ↓
One Alloy Pod per eligible node
```

---

# 21. Check Alloy Logs

Run:

```bash
kubectl logs -n alloy -l app.kubernetes.io/name=alloy
```

You should see Alloy discovering Kubernetes Pods.

For example:

```text
tailer running
```

and:

```text
opened log stream
```

You should not see:

```text
401 Unauthorized
no org id
```

---

# Part 3 — Deploy Nginx

# 22. Create Nginx Deployment

Create:

```bash
nano nginx.yaml
```

Use:

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

# 23. Verify Nginx Logs

Get the Pods:

```bash
kubectl get pods
```

Then:

```bash
kubectl logs <nginx-pod>
```

You should eventually see nginx access logs.

---

# 24. Generate Nginx Logs

Run:

```bash
kubectl exec <nginx-pod> -- curl localhost
```

Run multiple requests:

```bash
kubectl exec <nginx-pod> -- curl localhost
kubectl exec <nginx-pod> -- curl localhost
kubectl exec <nginx-pod> -- curl localhost
```

Verify:

```bash
kubectl logs <nginx-pod>
```

You should see entries similar to:

```text
GET / HTTP/1.1
```

---

# Part 4 — Verify Alloy Collects the Logs

Check Alloy:

```bash
kubectl logs -n alloy -l app.kubernetes.io/name=alloy --tail=30
```

You should see entries similar to:

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

This confirms Alloy discovered the nginx Pod.

---

# Part 5 — Verify Loki Received Logs

Query Loki labels:

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

After logs have been ingested, you should receive:

```json
{
  "status": "success",
  "data": [
    "instance",
    "job",
    "service_name"
  ]
}
```

The exact labels may vary depending on the Alloy/Loki version and configuration.

Do not assume that labels such as:

```text
pod
namespace
container
```

will automatically appear in every configuration.

---

# Part 6 — Query Logs Directly from Loki

You can query Loki's API directly.

For example:

```bash
kubectl run curl-test \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- \
  curl \
  -H "X-Scope-OrgID: local" \
  "http://loki.loki.svc.cluster.local:3100/loki/api/v1/query?query=%7Bjob%3D~%22.%2B%22%7D"
```

If logs exist, Loki returns the matching streams.

---

# Part 7 — Add `loki.process`

Now we will introduce log processing.

The pipeline becomes:

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

# 25. Modify Alloy Configuration

Update `alloy-values.yaml`:

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
          loki.process.add_app.receiver,
        ]
      }

      loki.process "add_app" {
        stage.static_labels {
          values = {
            app = "nginx",
          }
        }

        forward_to = [
          loki.write.endpoint.receiver,
        ]
      }

      loki.write "endpoint" {
        endpoint {
          url       = "http://loki.loki.svc.cluster.local:3100/loki/api/v1/push"
          tenant_id = "local"
        }
      }
```

---

# 26. Understand `loki.process`

The new component is:

```alloy
loki.process "add_app" {
```

It receives logs from:

```alloy
loki.source.kubernetes
```

using:

```alloy
loki.process.add_app.receiver
```

The process stage is:

```alloy
stage.static_labels {
  values = {
    app = "nginx",
  }
}
```

This adds:

```text
app="nginx"
```

to the log stream.

The logs are then forwarded to:

```alloy
loki.write.endpoint.receiver
```

---

# 27. Upgrade Alloy

Run:

```bash
helm upgrade alloy \
  grafana/alloy \
  -f alloy-values.yaml \
  -n alloy
```

Restart the Alloy controller if required by the deployment:

```bash
kubectl rollout restart daemonset alloy -n alloy
```

If Alloy is not a DaemonSet, inspect:

```bash
kubectl get deployment,statefulset,daemonset -n alloy
```

and restart the controller that Helm created.

---

# 28. Verify Alloy

```bash
kubectl get pods -n alloy
```

Then:

```bash
kubectl logs -n alloy -l app.kubernetes.io/name=alloy --tail=30
```

Make sure there are no configuration errors.

---

# 29. Generate New Nginx Logs

Run:

```bash
kubectl exec <nginx-pod> -- curl localhost
```

Run it several times:

```bash
kubectl exec <nginx-pod> -- curl localhost
kubectl exec <nginx-pod> -- curl localhost
kubectl exec <nginx-pod> -- curl localhost
```

---

# 30. Verify the New Label

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

You should now see:

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

The important addition is:

```text
app
```

---

# Part 8 — Install Grafana

# 31. Create Grafana Namespace

```bash
kubectl create namespace monitoring
```

---

# 32. Install Grafana

```bash
helm install grafana \
  grafana/grafana \
  -n monitoring
```

Check:

```bash
kubectl get pods -n monitoring
```

Wait until Grafana is:

```text
Running
```

---

# 33. Get Grafana Password

```bash
kubectl get secret grafana \
  -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

Username:

```text
admin
```

Save the password.

---

# 34. Access Grafana

Run:

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
Password: <password>
```

---

# Part 9 — Configure Loki Data Source

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

Because Grafana is running inside Kubernetes, it can resolve the Loki Service using Kubernetes DNS.

---

## Configure Tenant Header

Our Loki installation requires a tenant ID.

Under:

```text
HTTP Headers
```

Add:

```text
Header:

X-Scope-OrgID
```

Value:

```text
local
```

The final configuration is:

```text
URL:

http://loki.loki.svc.cluster.local:3100

HTTP Header:

X-Scope-OrgID: local
```

Click:

```text
Save & Test
```

Expected:

```text
Data source connected successfully
```

---

# Part 10 — Query Logs in Grafana

Go to:

```text
Explore
```

Select:

```text
Loki
```

---

## Query all logs

If your available labels support it:

```logql
{job=~".+"}
```

---

## Query the processed nginx logs

After the `loki.process` exercise:

```logql
{app="nginx"}
```

This selects streams with:

```text
app="nginx"
```

---

## Filter log content

For example:

```logql
{app="nginx"} |= "GET"
```

This means:

```text
Select streams where:

app = nginx

AND

log line contains "GET"
```

---

# Part 11 — Understanding Alloy Data Flow

Students should understand the following:

```text
discovery.kubernetes "pods"
```

creates:

```text
discovery.kubernetes.pods
```

Its exported target collection is:

```text
discovery.kubernetes.pods.targets
```

That becomes an argument to:

```text
loki.source.kubernetes "pods"
```

The source exposes:

```text
loki.source.kubernetes.pods.receiver
```

But we don't directly reference it here.

Instead, we configure:

```alloy
forward_to = [
  loki.process.add_app.receiver,
]
```

This sends log entries to the process component.

The process component exposes:

```text
loki.process.add_app.receiver
```

and forwards processed logs to:

```text
loki.write.endpoint.receiver
```

Finally:

```text
loki.write
```

sends the logs to Loki.

---

# Component Flow

```text
                    ARGUMENT
                       │
                       ▼
discovery.kubernetes.pods.targets
                       │
                       ▼
              loki.source.kubernetes
                       │
                       │ forward_to
                       ▼
              loki.process.add_app
                       │
                       │ forward_to
                       ▼
                loki.write.endpoint
                       │
                       ▼
                      Loki
```

---

# Part 12 — Arguments, Receivers and Exports

This lab demonstrates three important Alloy concepts.

## Arguments

Example:

```alloy
targets = discovery.kubernetes.pods.targets
```

`targets` is an argument accepted by:

```text
loki.source.kubernetes
```

---

## Receivers

Example:

```alloy
loki.write.endpoint.receiver
```

This identifies the receiver exposed by:

```alloy
loki.write "endpoint"
```

Another example:

```alloy
loki.process.add_app.receiver
```

---

## Exports

Components can expose data that other components can reference.

For example:

```text
discovery.kubernetes.pods.targets
```

is an exported target collection.

It is consumed by:

```alloy
targets = discovery.kubernetes.pods.targets
```

---

# Part 13 — Troubleshooting

## Problem 1 — Loki installation fails with replica validation error

If you see:

```text
You have more than zero replicas configured for both
the monolithic and simple scalable targets
```

verify that all unused deployment targets have:

```yaml
replicas: 0
```

and that:

```yaml
deploymentMode: Monolithic
```

is configured.

Check:

```bash
helm get values loki -n loki
```

---

# Problem 2 — Loki Pod is not ready

Check:

```bash
kubectl get pods -n loki
```

Then:

```bash
kubectl describe pod loki-0 -n loki
```

Check logs:

```bash
kubectl logs -n loki loki-0
```

If Loki has multiple containers, use:

```bash
kubectl logs -n loki loki-0 -c <container-name>
```

---

# Problem 3 — Loki DNS doesn't work

Check:

```bash
kubectl get svc -n loki
```

You should have:

```text
loki
```

Test DNS/networking:

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

# Problem 4 — Loki returns `401 no org id`

If:

```bash
curl http://loki.loki.svc.cluster.local:3100/loki/api/v1/labels
```

returns:

```text
401 Unauthorized
no org id
```

Loki requires a tenant.

Use:

```bash
curl \
  -H "X-Scope-OrgID: local" \
  http://loki.loki.svc.cluster.local:3100/loki/api/v1/labels
```

And ensure Alloy has:

```alloy
tenant_id = "local"
```

Grafana must also send:

```text
X-Scope-OrgID: local
```

---

# Problem 5 — Alloy returns `401 no org id`

Check the ConfigMap:

```bash
kubectl get configmap alloy -n alloy -o yaml
```

Make sure it contains:

```alloy
tenant_id = "local"
```

Then restart the Alloy controller.

For a DaemonSet:

```bash
kubectl rollout restart daemonset alloy -n alloy
```

Check:

```bash
kubectl get pods -n alloy
```

Then:

```bash
kubectl logs -n alloy -l app.kubernetes.io/name=alloy
```

---

# Problem 6 — Alloy cannot discover Pods

Check Alloy logs:

```bash
kubectl logs -n alloy -l app.kubernetes.io/name=alloy
```

Look for:

```text
forbidden
permission denied
cannot list pods
```

Check RBAC:

```bash
kubectl get clusterrole | grep alloy
```

and:

```bash
kubectl get clusterrolebinding | grep alloy
```

Also inspect the Alloy ServiceAccount:

```bash
kubectl get serviceaccount -n alloy
```

---

# Problem 7 — Alloy discovers Pods but no logs appear

First verify Kubernetes itself has logs:

```bash
kubectl logs <nginx-pod>
```

If logs exist:

```text
Kubernetes
    ↓
    OK
```

Then check:

```text
Kubernetes
    ↓
Alloy
```

using:

```bash
kubectl logs -n alloy -l app.kubernetes.io/name=alloy
```

Then verify Loki:

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

---

# Problem 8 — Grafana cannot connect to Loki

Verify Loki:

```bash
kubectl get pods -n loki
```

Verify Service:

```bash
kubectl get svc -n loki
```

Test:

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

Then check the Grafana Loki data source.

URL:

```text
http://loki.loki.svc.cluster.local:3100
```

Header:

```text
X-Scope-OrgID: local
```

---

# Part 14 — Cleanup

Remove nginx:

```bash
kubectl delete -f nginx.yaml
```

Remove Grafana:

```bash
helm uninstall grafana -n monitoring
```

Remove Alloy:

```bash
helm uninstall alloy -n alloy
```

Remove Loki:

```bash
helm uninstall loki -n loki
```

Remove namespaces:

```bash
kubectl delete namespace monitoring
kubectl delete namespace alloy
kubectl delete namespace loki
```

---

# 🎯 Final Learning Outcome

At the end of the lab, students should be able to explain:

```text
How does a Kubernetes Pod's log reach Grafana?
```

The answer should be:

```text
Application Pod
      ↓
Container stdout/stderr
      ↓
Kubernetes logs
      ↓
Grafana Alloy
      ↓
discovery.kubernetes
      ↓
loki.source.kubernetes
      ↓
loki.process
      ↓
loki.write
      ↓
Loki
      ↓
LogQL
      ↓
Grafana Explore
```

They should also understand that Alloy is not the log storage system.

```text
Alloy
  =
Collect + Process + Forward

Loki
  =
Store + Query

Grafana
  =
Visualize + Explore
```

---

# 🧠 Key Alloy Concepts Demonstrated

| Concept        | Used in Lab                                                                    |
| -------------- | ------------------------------------------------------------------------------ |
| Component      | `discovery.kubernetes`, `loki.source.kubernetes`, `loki.process`, `loki.write` |
| Component name | `"pods"`, `"add_app"`, `"endpoint"`                                            |
| Argument       | `targets`, `forward_to`, `url`, `tenant_id`                                    |
| Export         | `discovery.kubernetes.pods.targets`                                            |
| Receiver       | `loki.write.endpoint.receiver`                                                 |
| Data flow      | `SOURCE → PROCESS → WRITE`                                                     |
| Discovery      | Kubernetes Pods                                                                |
| Source         | Kubernetes container logs                                                      |
| Processing     | Add `app="nginx"`                                                              |
| Destination    | Loki                                                                           |
| Query          | LogQL                                                                          |
| Visualization  | Grafana Explore                                                                |
