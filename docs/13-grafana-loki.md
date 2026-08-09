# 13. Grafana + Loki

Grafana provides the interface for querying and visualizing Loki logs.

## Architecture

```text
Application
    ↓
Alloy
    ↓
Loki
    ↓
Grafana
```

## Add Loki as a Data Source

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

If Loki is running locally:

```text
http://localhost:3100
```

If Grafana is another container in the same Docker Compose network:

```text
http://loki:3100
```

## Explore Logs

Open:

```text
Explore
```

Select:

```text
Loki
```

Run:

```logql
{job="app"}
```

## Search Errors

```logql
{job="app"} |= "error"
```

## JSON Logs

```logql
{job="app"} | json | level="error"
```

## Metrics + Logs

```text
Prometheus ──→ Metrics ──┐
                         ├──→ Grafana
Loki ────────→ Logs ─────┘
```

Metrics tell you that something is wrong; logs can provide the detailed event information needed to investigate why.
