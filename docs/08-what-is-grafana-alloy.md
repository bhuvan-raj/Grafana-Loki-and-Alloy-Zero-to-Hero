# 8. What is Grafana Alloy?

Grafana Alloy is an **open-source telemetry collector**.

It can collect, process, and forward:

- Logs
- Metrics
- Traces
- Profiles

In this repository, the main focus is logs.

## Why Alloy?

A logging system needs a collector.

```text
Log File
   ↓
Alloy
   ↓
Loki
```

Alloy can:

1. Collect data
2. Process data
3. Add or change metadata
4. Send data to a destination

## Alloy + Loki

```text
Application
    ↓
Log
    ↓
Grafana Alloy
    ↓
Loki
    ↓
Grafana
```

## Alloy vs Promtail

Promtail reached end of life on **March 2, 2026**.

Grafana's future development for this logging pipeline is centered on Alloy.

Therefore:

```text
Old:
Promtail → Loki

Modern:
Alloy → Loki
```

## Alloy is Broader Than Logging

Alloy can build pipelines for:

```text
Metrics
Logs
Traces
Profiles
```

It can therefore act as a unified telemetry collector.
