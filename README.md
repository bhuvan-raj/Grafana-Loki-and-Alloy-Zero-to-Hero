<div align="center">

# 📜 Grafana Loki & Alloy — Zero to Hero

**A practical, hands-on repository for learning centralized logging — from fundamentals to real-world Kubernetes deployments.**

![Status](https://img.shields.io/badge/status-active-brightgreen)
![Topics](https://img.shields.io/badge/topics-18-blue)
![Level](https://img.shields.io/badge/level-beginner%20to%20advanced-orange)
![Stack](https://img.shields.io/badge/stack-loki%20%2B%20alloy%20%2B%20grafana-red)

</div>

---

## 🧭 About This Repo

This repository covers **Grafana Loki** and **Grafana Alloy** end to end — log fundamentals, Loki's architecture and query language, Alloy's configuration and pipelines, and hands-on labs across Docker and Kubernetes.

> ⚠️ **Important:** Promtail reached end of life on March 2, 2026. This repository teaches **Grafana Alloy** as the primary log collector instead of Promtail.

---

## 🏗️ Architecture

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

---

## 📚 Table of Contents

### 🪵 Loki Fundamentals

| # | Topic | Description |
|---|-------|-------------|
| 1 | [Introduction to Logging](docs/01-introduction-to-logging.md) | Why centralized logging matters |
| 2 | [What is Grafana Loki?](docs/02-what-is-loki.md) | Core concepts and philosophy |
| 3 | [Loki Architecture](docs/03-loki-architecture.md) | Components and data flow |
| 4 | [Loki Components](docs/04-loki-components.md) | Distributor, ingester, querier, and more |
| 5 | [Labels, Streams and Structured Metadata](docs/05-labels-streams-metadata.md) | How Loki indexes and organizes logs |
| 6 | [LogQL](docs/06-logql.md) | Loki's query language |
| 7 | [Loki Storage and Deployment Modes](docs/07-loki-storage-and-deployment-modes.md) | Monolithic, simple-scalable, microservices |

### 🛰️ Grafana Alloy

| # | Topic | Description |
|---|-------|-------------|
| 8 | [What is Grafana Alloy?](docs/08-what-is-grafana-alloy.md) | Alloy's role as a telemetry collector |
| 9 | [Alloy Architecture and Components](docs/09-alloy-architecture-and-components.md) | Pipelines, components, and wiring |
| 10 | [Alloy Configuration Language](docs/10-alloy-configuration.md) | Writing Alloy configs |
| 11 | [Collecting and Processing Logs with Alloy](docs/11-alloy-log-collection-and-processing.md) | Discovery, scraping, and processing |
| 12 | [Alloy to Loki](docs/12-alloy-to-loki.md) | Shipping logs into Loki |

### 📊 Grafana and Loki

| # | Topic | Description |
|---|-------|-------------|
| 13 | [Grafana + Loki](docs/13-grafana-loki.md) | Exploring and visualizing logs |
| 14 | [Troubleshooting](docs/14-troubleshooting.md) | Common issues and fixes |
| 15 | [Production Best Practices](docs/15-production-best-practices.md) | Running the stack reliably at scale |

### 🧪 Hands-on Labs

| # | Lab | Description |
|---|-----|-------------|
| 16 | [Docker Compose: Loki + Alloy + Grafana](labs/01-docker-compose-loki-alloy-grafana.md) | Full local stack in minutes |
| 17 | [Linux Log File → Alloy → Loki](labs/02-linux-log-file-to-loki.md) | Tail and ship a real log file |
| 18 | [Kubernetes Logging with Alloy](labs/03-kubernetes-logging-with-alloy.md) | Cluster-wide log collection |

---

## 🗺️ Learning Path

```text
 1. Logging Fundamentals
        ↓
 2. Loki
        ↓
 3. Loki Architecture
        ↓
 4. Labels & Streams
        ↓
 5. LogQL
        ↓
 6. Alloy
        ↓
 7. Alloy Components
        ↓
 8. Alloy Configuration
        ↓
 9. Collect → Process → Write
        ↓
10. Grafana
        ↓
11. Docker
        ↓
12. Kubernetes
        ↓
13. Production Logging
```

---

## 📁 Repository Structure

```text
.
├── README.md
├── assets/
├── docs/
└── labs/
```

---

## 🔭 Observability Stack

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

---

## 📖 Official Documentation

| Resource | Link |
|----------|------|
| Loki Docs | [grafana.com/docs/loki](https://grafana.com/docs/loki/) |
| Alloy Docs | [grafana.com/docs/alloy](https://grafana.com/docs/alloy/) |
| Grafana Docs | [grafana.com/docs/grafana](https://grafana.com/docs/grafana/) |

---

<div align="center">

**⭐ If this helped you, consider starring the repo!**

</div>
