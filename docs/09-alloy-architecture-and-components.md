# 9. Alloy Architecture and Components

Grafana Alloy uses a **component-based architecture**. Instead of configuring one large monolithic agent, you build a telemetry pipeline by connecting multiple small components together.

Each component has a specific responsibility, such as:

* Discovering targets
* Collecting logs
* Collecting metrics
* Receiving traces
* Processing or transforming telemetry
* Filtering unwanted data
* Adding metadata
* Forwarding telemetry to a backend

The most important concept to understand is:

> **Alloy collects telemetry, processes it through a pipeline, and forwards it to an observability backend.**

---

## 9.1 Basic Pipeline Model

A simple Alloy pipeline can be represented as:

```text
SOURCE
  │
  ▼
PROCESS
  │
  ▼
WRITE
  │
  ▼
BACKEND
```

For example, for a log file:

```text
Log File
   │
   ▼
loki.source.file
   │
   ▼
loki.process
   │
   ▼
loki.write
   │
   ▼
Loki
```

Each component performs a different job:

| Component          | Responsibility                                  |
| ------------------ | ----------------------------------------------- |
| `loki.source.file` | Reads logs from a file                          |
| `loki.process`     | Processes, parses, filters, and transforms logs |
| `loki.write`       | Sends logs to Loki                              |
| Loki               | Stores and queries the logs                     |

This gives us the fundamental Alloy mental model:

```text
Collect → Transform → Send
```

---

# 9.2 What Is a Component?

A **component** is an individual building block in an Alloy configuration.

For example:

```alloy
loki.source.file "app" {
    targets = [
        {
            __path__ = "/var/log/app.log",
        },
    ]

    forward_to = [
        loki.write.default.receiver,
    ]
}
```

Here:

```text
loki.source.file
```

is the **component type**.

```text
"app"
```

is the **component label**.

So:

```text
loki.source.file "app"
```

means:

> Create a `loki.source.file` component with the label `app`.

Components can expose outputs that other components can consume.

For example:

```text
loki.source.file.app
        │
        │ logs
        ▼
loki.process.app
```

This connection is created using a **component reference**.

---

# 9.3 Component Types

Alloy contains many different component types.

Some components are responsible for collecting telemetry:

```text
loki.source.file
loki.source.docker
loki.source.kubernetes
prometheus.scrape
otelcol.receiver.otlp
```

Some process telemetry:

```text
loki.process
prometheus.relabel
otelcol.processor.batch
```

Some send telemetry to external systems:

```text
loki.write
prometheus.remote_write
otelcol.exporter.otlp
```

There are also components for:

```text
Service discovery
Relabeling
Authentication
Filtering
Debugging
Secrets
Storage
Clustering
```

Therefore, Alloy is better understood as a **telemetry pipeline framework** rather than simply a log collector.

---

# 9.4 Source Components

Source components are responsible for **collecting or receiving telemetry**.

For logs, some commonly used components are:

```text
loki.source.file
loki.source.docker
loki.source.kubernetes
loki.source.journal
```

The source component is normally the starting point of a log pipeline.

For example:

```text
Application
    │
    ▼
Log file
    │
    ▼
loki.source.file
```

The source reads the logs and produces log entries that can be forwarded to another component.

---

# 9.5 `loki.source.file`

`loki.source.file` reads log entries from files on the filesystem.

For example:

```text
/var/log/app.log
```

A simplified configuration is:

```alloy
loki.source.file "app" {
    targets = [
        {
            __path__ = "/var/log/app.log",
        },
    ]

    forward_to = [
        loki.write.default.receiver,
    ]
}
```

The flow is:

```text
/var/log/app.log
       │
       ▼
loki.source.file
       │
       │ log entries
       ▼
loki.write
       │
       ▼
Loki
```

In a real environment, the source can monitor files and continue reading new log entries as they are written.

---

# 9.6 `loki.source.docker`

When applications are running as Docker containers, their logs are usually managed by Docker's logging system.

Alloy provides:

```text
loki.source.docker
```

to collect Docker container logs.

The architecture becomes:

```text
Docker Container
       │
       │ stdout/stderr
       ▼
Docker logging
       │
       ▼
loki.source.docker
       │
       ▼
loki.process
       │
       ▼
loki.write
       │
       ▼
Loki
```

This is useful when Alloy is running on a Docker host and needs to collect logs from containers.

---

# 9.7 `loki.source.kubernetes`

For Kubernetes environments, Alloy provides:

```text
loki.source.kubernetes
```

This component can collect logs from Kubernetes Pods.

A simplified architecture is:

```text
Kubernetes Pod
      │
      │ container logs
      ▼
Kubernetes
      │
      ▼
loki.source.kubernetes
      │
      ▼
loki.process
      │
      ▼
loki.write
      │
      ▼
Loki
```

The important thing to understand is that Kubernetes log collection normally involves **discovering which Pods exist and determining which logs should be collected**.

In production Kubernetes configurations, Alloy commonly works together with Kubernetes discovery components.

For example:

```text
Kubernetes API
      │
      ▼
discovery.kubernetes
      │
      │ discovered targets
      ▼
loki.source.kubernetes
      │
      │ logs
      ▼
loki.process
      │
      ▼
loki.write
      │
      ▼
Loki
```

So there are two separate concepts:

```text
Discovery
   ↓
"What should I collect?"

Source
   ↓
"Collect the telemetry."
```

This distinction is extremely important when working with Alloy on Kubernetes.

---

# 9.8 `loki.source.journal`

Linux systems using `systemd` store many system logs in the **systemd journal**.

Alloy can collect these logs using:

```text
loki.source.journal
```

The architecture becomes:

```text
systemd journal
      │
      ▼
loki.source.journal
      │
      ▼
loki.process
      │
      ▼
loki.write
      │
      ▼
Loki
```

This is particularly useful when monitoring Linux servers and system services.

---

# 9.9 Processing Components

After collecting telemetry, we often need to transform it before sending it to the backend.

For Loki pipelines, the primary processing component is:

```text
loki.process
```

It acts as a processing pipeline for log entries.

For example:

```text
Log
 │
 ▼
loki.process
 │
 ├── Parse
 ├── Extract
 ├── Filter
 ├── Modify
 └── Add metadata
 │
 ▼
Next component
```

---

# 9.10 What Can `loki.process` Do?

`loki.process` can perform many operations on incoming logs.

### Parse logs

Suppose the application produces:

```text
2026-08-16 18:30:21 ERROR payment failed
```

We may want to extract:

```text
timestamp
level
message
```

The processing pipeline can parse the log.

---

### Extract fields

Suppose an application generates JSON:

```json
{
  "level": "error",
  "service": "payment",
  "user": "bubu",
  "message": "payment failed"
}
```

The processing stage can extract fields such as:

```text
level = error
service = payment
user = bubu
message = payment failed
```

These extracted values can then be used for further processing.

---

### Filter logs

Suppose you only want error logs.

```text
INFO
INFO
DEBUG
ERROR
INFO
ERROR
```

The processing stage can filter them:

```text
INFO     ──X
INFO     ──X
DEBUG    ──X
ERROR    ──►
INFO     ──X
ERROR    ──►
```

The result sent to Loki contains only the required logs.

---

### Add labels

Processing can also be used to create or modify labels.

For example:

```text
service=payment
environment=production
```

Then Grafana/Loki can query:

```text
{service="payment", environment="production"}
```

However, **be careful when creating labels**.

High-cardinality values such as:

```text
request_id
user_id
transaction_id
```

generally should not blindly become Loki labels because this can create excessive stream cardinality.

A good Loki design is to use labels for relatively stable dimensions such as:

```text
namespace
pod
container
service
environment
```

while keeping highly variable values inside the log body or structured metadata where appropriate.

---

# 9.11 Processing Is Not the Same as Collection

This distinction is important.

The source:

```text
loki.source.file
```

answers:

> Where do the logs come from?

The processor:

```text
loki.process
```

answers:

> What should I do with those logs?

The writer:

```text
loki.write
```

answers:

> Where should the processed logs go?

Therefore:

```text
SOURCE
  │
  │ "Get the logs"
  ▼
PROCESS
  │
  │ "Transform the logs"
  ▼
WRITE
  │
  │ "Send the logs"
  ▼
LOKI
```

---

# 9.12 Write Components

Once logs have been collected and processed, they need to be sent somewhere.

For Loki, the component responsible for this is:

```text
loki.write
```

For example:

```alloy
loki.write "default" {

    endpoint {
        url = "http://loki:3100/loki/api/v1/push"
    }
}
```

The purpose of this component is to send log entries to the configured Loki endpoint.

The overall pipeline becomes:

```text
Application
     │
     ▼
Log source
     │
     ▼
loki.process
     │
     ▼
loki.write
     │
     ▼
Loki
```

---

# 9.13 Component References

One of the most important Alloy concepts is the **component reference**.

Consider:

```alloy
loki.write "default" {
    endpoint {
        url = "http://loki:3100/loki/api/v1/push"
    }
}
```

This component exposes a receiver:

```text
loki.write.default.receiver
```

Another component can send logs to that receiver:

```alloy
loki.source.file "app" {
    targets = [
        {
            __path__ = "/var/log/app.log",
        },
    ]

    forward_to = [
        loki.write.default.receiver,
    ]
}
```

The important part is:

```text
loki.write.default.receiver
```

Break it down:

```text
loki.write
    │
    └── Component type

default
    │
    └── Component label

receiver
    │
    └── Export/connection point
```

So Alloy components are connected through their exported values and receivers.

---

# 9.14 A Complete Simple Pipeline

Putting everything together:

```alloy
loki.source.file "app" {
    targets = [
        {
            __path__ = "/var/log/app.log",
        },
    ]

    forward_to = [
        loki.process.app.receiver,
    ]
}

loki.process "app" {

    stage.regex {
        expression = "level=(?P<level>\\w+)"
    }

    stage.labels {
        values = {
            level = "",
        }
    }

    forward_to = [
        loki.write.default.receiver,
    ]
}

loki.write "default" {

    endpoint {
        url = "http://loki:3100/loki/api/v1/push"
    }
}
```

The pipeline is:

```text
                 /var/log/app.log
                        │
                        ▼
              ┌───────────────────┐
              │ loki.source.file  │
              │                   │
              │ Collect logs      │
              └─────────┬─────────┘
                        │
                        ▼
              ┌───────────────────┐
              │   loki.process    │
              │                   │
              │ Parse             │
              │ Extract           │
              │ Transform         │
              │ Filter            │
              └─────────┬─────────┘
                        │
                        ▼
              ┌───────────────────┐
              │    loki.write     │
              │                   │
              │ Send to Loki      │
              └─────────┬─────────┘
                        │
                        ▼
                     Loki
```

---

# 9.15 Alloy Pipelines Can Be More Complex

The basic model:

```text
SOURCE
   ↓
PROCESS
   ↓
WRITE
```

is useful for learning, but real production pipelines can be much more complex.

For example:

```text
                         ┌──► Process A ──► Loki
                         │
Source ──► Processing ───┤
                         │
                         └──► Process B ──► Loki
```

You can also have multiple sources:

```text
File ───────────────┐
                    │
Docker ─────────────┼──► Processing ──► Loki
                    │
Kubernetes ─────────┘
```

Or multiple destinations:

```text
                    ┌──► Loki
                    │
Source ─► Process ──┤
                    │
                    └──► Another backend
```

This is why Alloy is better thought of as a **pipeline graph** rather than a simple linear agent.

---

# 9.16 Fan-In and Fan-Out

### Fan-In

Multiple sources can feed one processing pipeline:

```text
File ─────────┐
              │
Docker ───────┼──► Process ──► Loki
              │
Kubernetes ───┘
```

This is called **fan-in**.

---

### Fan-Out

One telemetry stream can be sent to multiple destinations:

```text
                 ┌──► Loki
                 │
Source ─► Process┤
                 │
                 └──► Another destination
```

This is called **fan-out**.

These patterns become useful when designing production observability architectures.

---

# 9.17 Discovery Components

In dynamic environments such as Kubernetes, you often don't want to manually configure every target.

Imagine:

```text
Pod A
Pod B
Pod C
Pod D
```

Tomorrow:

```text
Pod A
Pod B
Pod C
Pod D
Pod E
Pod F
```

You don't want to manually modify Alloy every time a Pod appears.

This is where **discovery components** become important.

For Kubernetes, Alloy can use:

```text
discovery.kubernetes
```

to discover Kubernetes resources.

The architecture becomes:

```text
              Kubernetes API
                    │
                    ▼
         discovery.kubernetes
                    │
              discovered
               targets
                    │
                    ▼
        Kubernetes log source
                    │
                    ▼
             loki.process
                    │
                    ▼
               loki.write
                    │
                    ▼
                  Loki
```

So remember:

```text
Discovery = Find targets
Source    = Collect telemetry
Process   = Transform telemetry
Write     = Send telemetry
```

---

# 9.18 Metrics Follow the Same Concept

The same architecture exists for metrics.

Instead of:

```text
loki.source
   ↓
loki.process
   ↓
loki.write
```

you may have:

```text
prometheus.scrape
        ↓
prometheus.relabel
        ↓
prometheus.remote_write
        ↓
Mimir / Prometheus-compatible backend
```

Conceptually:

```text
Metrics Target
      │
      ▼
Collect
      │
      ▼
Process
      │
      ▼
Forward
      │
      ▼
Metrics Backend
```

So Alloy isn't fundamentally a "Loki tool".

The same pipeline architecture can be applied to:

```text
Logs
Metrics
Traces
Profiles
```

---

# 9.19 Traces Follow the Same Concept

For OpenTelemetry pipelines, you might see:

```text
Application
    │
    │ OTLP
    ▼
otelcol.receiver.otlp
    │
    ▼
otelcol.processor.batch
    │
    ▼
otelcol.exporter.otlp
    │
    ▼
Tempo
```

Again:

```text
Receive
   ↓
Process
   ↓
Export
```

The component names are different because they belong to the OpenTelemetry component family, but the fundamental architecture remains the same.

---

# 9.20 Three Layers to Remember

When learning Alloy, think about the architecture in three logical layers.

## Layer 1 — Collection

```text
"Where does telemetry come from?"
```

Examples:

```text
loki.source.file
loki.source.kubernetes
loki.source.docker
prometheus.scrape
otelcol.receiver.otlp
```

---

## Layer 2 — Processing

```text
"What should happen to the telemetry?"
```

Examples:

```text
loki.process
prometheus.relabel
otelcol.processor.batch
```

Processing can include:

```text
Parse
Filter
Transform
Relabel
Enrich
Batch
Drop
```

---

## Layer 3 — Delivery

```text
"Where should the telemetry go?"
```

Examples:

```text
loki.write
prometheus.remote_write
otelcol.exporter.otlp
```

---

# 9.21 Complete Mental Model

The best mental model for Alloy is:

```text
                         GRAFANA ALLOY

       DISCOVERY
           │
           │ Find targets
           ▼
       COLLECTION
           │
           │ Collect telemetry
           ▼
       PROCESSING
           │
           │ Parse / Filter / Transform
           ▼
        FORWARDING
           │
           │ Send telemetry
           ▼
       OBSERVABILITY
        BACKEND
```

For logs:

```text
Discovery
    ↓
Log Source
    ↓
loki.process
    ↓
loki.write
    ↓
Loki
```

For metrics:

```text
Discovery
    ↓
prometheus.scrape
    ↓
prometheus.relabel
    ↓
prometheus.remote_write
    ↓
Metrics backend
```

For traces:

```text
OTLP Receiver
    ↓
Processor
    ↓
OTLP Exporter
    ↓
Tempo
```

---

# 9.22 Important Terminology

| Term                    | Meaning                                                               |
| ----------------------- | --------------------------------------------------------------------- |
| **Component**           | Individual building block in Alloy                                    |
| **Source**              | Collects or receives telemetry                                        |
| **Discovery**           | Finds targets/resources dynamically                                   |
| **Processor**           | Modifies, filters, parses, or transforms telemetry                    |
| **Receiver**            | An input connection point exposed by a component                      |
| **Export**              | Data/value exposed by a component for another component to consume    |
| **Writer**              | Sends telemetry to a backend                                          |
| **Pipeline**            | Connected sequence of Alloy components                                |
| **Component reference** | Used to connect one component to another                              |
| **Backend**             | System that stores/processes telemetry, such as Loki, Mimir, or Tempo |

---

# 9.23 The Golden Rule

Whenever you see an Alloy configuration, don't immediately try to memorize the syntax.

Instead, trace the data:

```text
Where does the data come from?
          ↓
How is it discovered?
          ↓
Which component collects it?
          ↓
How is it processed?
          ↓
Where is it forwarded?
          ↓
Which backend stores it?
```

For a Kubernetes Loki deployment, you should be able to mentally trace:

```text
Kubernetes API
      │
      ▼
discovery.kubernetes
      │
      ▼
loki.source.kubernetes
      │
      ▼
loki.process
      │
      ▼
loki.write
      │
      ▼
Loki
      │
      ▼
Grafana
```

Once you understand this flow, **Alloy configuration becomes much easier to read and troubleshoot**.
