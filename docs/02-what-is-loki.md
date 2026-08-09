# 2. What is Grafana Loki?

Grafana Loki is a **log aggregation system** designed to store and query logs efficiently.

The simple comparison is:

```text
Prometheus → Metrics
Loki       → Logs
```

## How Loki Works

Loki does not index the full contents of every log line.

Instead, it indexes **labels** that identify log streams.

Example:

```text
{job="nginx", environment="production"}
```

The log contents are stored separately.

## Log Stream

A log stream is a collection of log entries that share the same set of labels.

## Why This Design?

Indexing every field in every log can become expensive at scale.

Loki keeps the indexed data smaller by using labels to identify streams.

## Loki Pipeline

```text
Application
    ↓
Alloy
    ↓
Loki
    ↓
LogQL
    ↓
Grafana
```

## Loki vs Prometheus

| Prometheus | Loki |
|---|---|
| Metrics | Logs |
| Time series | Log streams |
| PromQL | LogQL |
| Scrapes metrics | Receives log data |
| Metric labels | Stream labels |

Loki is not simply a text-file search engine. It first selects streams using labels and then filters log content.
