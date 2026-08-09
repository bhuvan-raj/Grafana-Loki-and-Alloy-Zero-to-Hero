# Lab 1 — Loki + Alloy + Grafana with Docker Compose

This lab creates a complete local logging stack:

```text
Log File → Alloy → Loki → Grafana
```

## Prerequisites

- Docker
- Docker Compose

## Directory

```text
loki-alloy-lab/
├── docker-compose.yml
├── alloy/
│   └── config.alloy
└── logs/
    └── app.log
```

## 1. Create the Directory

```bash
mkdir -p loki-alloy-lab/alloy
mkdir -p loki-alloy-lab/logs
cd loki-alloy-lab
touch logs/app.log
```

## 2. Create `docker-compose.yml`

```yaml
services:
  loki:
    image: grafana/loki:latest
    command: -config.file=/etc/loki/local-config.yaml
    ports:
      - "3100:3100"

  alloy:
    image: grafana/alloy:latest
    command:
      - run
      - /etc/alloy/config.alloy
    volumes:
      - ./alloy/config.alloy:/etc/alloy/config.alloy:ro
      - ./logs:/logs:ro
    depends_on:
      - loki

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    depends_on:
      - loki
```

## 3. Create `alloy/config.alloy`

```alloy
loki.source.file "app" {
  targets = [
    {
      "__path__" = "/logs/app.log",
      "job"     = "app",
    },
  ]

  forward_to = [loki.write.local.receiver]
}

loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

## 4. Start the Stack

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

## 5. Generate Logs

```bash
echo 'INFO application started' >> logs/app.log
echo 'INFO user logged in' >> logs/app.log
echo 'ERROR database connection failed' >> logs/app.log
```

## 6. Check Alloy

```bash
docker compose logs alloy
```

## 7. Check Loki

```bash
curl http://localhost:3100/ready
```

## 8. Open Grafana

```text
http://localhost:3000
```

Add Loki as a data source.

Use:

```text
http://loki:3100
```

because Grafana reaches Loki through the Docker Compose network.

## 9. Query Logs

In **Explore**, select Loki:

```logql
{job="app"}
```

Search errors:

```logql
{job="app"} |= "ERROR"
```

## Expected Result

```text
app.log
   ↓
Alloy
   ↓
Loki
   ↓
Grafana Explore
```
