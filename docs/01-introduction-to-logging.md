# 1. Introduction to Logging

## What is a Log?

A log is a record of an event produced by an application, server, database, container, or other system.

Example:

```text
2026-08-09 10:30:12 INFO User login successful user=bubu
```

## Why Do We Need Logs?

Logs help answer:

- What happened?
- When did it happen?
- Which application produced it?
- Why did an error occur?
- Which request or user was affected?

## The Problem with Local Logs

```text
Server 1 ── /var/log/app.log
Server 2 ── /var/log/app.log
Server 3 ── /var/log/app.log
```

Searching many servers manually is difficult.

## Centralized Logging

```text
Server 1 ─┐
Server 2 ─┤
Server 3 ─┼──→ Collector ──→ Loki ──→ Grafana
Server 4 ─┘
```

## Metrics vs Logs

| Metrics | Logs |
|---|---|
| Numeric measurements | Event records |
| Good for trends | Good for details |
| CPU = 80% | "Database connection failed" |
| Prometheus | Loki |

## Logging Pipeline

```text
Application
    ↓
Log File / Container Log
    ↓
Grafana Alloy
    ↓
Loki
    ↓
LogQL
    ↓
Grafana
```
