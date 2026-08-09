# Grafana Loki & Alloy — Zero to Hero

A practical **Grafana Loki and Grafana Alloy Zero to Hero** repository for learning centralized logging from fundamentals to real-world Kubernetes deployments.

## Architecture

```text
Applications / Servers
        │
        │ Logs
        ▼
  Grafana Alloy
        │
        │ Push
        ▼
      Loki
        │
        │ LogQL
        ▼
     Grafana
        │
        ▼
    Dashboards
```

> **Important:** Promtail reached end of life on March 2, 2026. This repository teaches Grafana Alloy as the primary log collector instead of Promtail.

## Table of Contents

### Loki Fundamentals

1. [Introduction to Logging](docs/01-introduction-to-logging.md)
2. [What is Grafana Loki?](docs/02-what-is-loki.md)
3. [Loki Architecture](docs/03-loki-architecture.md)
4. [Loki Components](docs/04-loki-components.md)
5. [Labels, Streams and Structured Metadata](docs/05-labels-streams-metadata.md)
6. [LogQL](docs/06-logql.md)
7. [Loki Storage and Deployment Modes](docs/07-loki-storage-and-deployment-modes.md)

### Grafana Alloy

8. [What is Grafana Alloy?](docs/08-what-is-grafana-alloy.md)
9. [Alloy Architecture and Components](docs/09-alloy-architecture-and-components.md)
10. [Alloy Configuration Language](docs/10-alloy-configuration.md)
11. [Collecting and Processing Logs with Alloy](docs/11-alloy-log-collection-and-processing.md)
12. [Alloy to Loki](docs/12-alloy-to-loki.md)

### Grafana and Loki

13. [Grafana + Loki](docs/13-grafana-loki.md)
14. [Troubleshooting](docs/14-troubleshooting.md)
15. [Production Best Practices](docs/15-production-best-practices.md)

### Hands-on Labs

16. [Docker Compose: Loki + Alloy + Grafana](labs/01-docker-compose-loki-alloy-grafana.md)
17. [Linux Log File → Alloy → Loki](labs/02-linux-log-file-to-loki.md)
18. [Kubernetes Logging with Alloy](labs/03-kubernetes-logging-with-alloy.md)

## Learning Path

```text
Logging Fundamentals
        ↓
      Loki
        ↓
 Loki Architecture
        ↓
Labels & Streams
        ↓
     LogQL
        ↓
    Alloy
        ↓
Alloy Components
        ↓
Alloy Configuration
        ↓
Collect → Process → Write
        ↓
      Grafana
        ↓
      Docker
        ↓
   Kubernetes
        ↓
Production Logging
```

## Repository Structure

```text
.
├── README.md
├── assets/
├── docs/
└── labs/
```

## Observability Stack

```text
                  OBSERVABILITY
                       │
        ┌──────────────┼──────────────┐
        │              │              │
      Metrics         Logs          Traces
        │              │              │
   Prometheus         Loki          Tempo
        │              │              │
        └──────────────┼──────────────┘
                       │
                    Grafana
```

## Official Documentation

- https://grafana.com/docs/loki/
- https://grafana.com/docs/alloy/
- https://grafana.com/docs/grafana/
