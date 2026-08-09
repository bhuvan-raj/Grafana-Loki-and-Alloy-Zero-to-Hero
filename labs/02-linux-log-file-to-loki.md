# Lab 2 — Linux Log File → Alloy → Loki

This lab teaches how Alloy reads a Linux log file and sends it to Loki.

## Architecture

```text
Linux Server
    │
    │ /opt/logs/app.log
    ▼
Grafana Alloy
    │
    ▼
Loki
    │
    ▼
Grafana
```

## Prerequisites

- Linux machine
- Alloy installed
- Reachable Loki instance
- Grafana connected to Loki

## 1. Create a Test Log

```bash
sudo mkdir -p /opt/logs
sudo touch /opt/logs/app.log
```

Generate logs:

```bash
echo "$(date) INFO application started" | sudo tee -a /opt/logs/app.log
echo "$(date) INFO request received" | sudo tee -a /opt/logs/app.log
echo "$(date) ERROR database connection failed" | sudo tee -a /opt/logs/app.log
```

## 2. Configure Alloy

Create `config.alloy`:

```alloy
loki.source.file "app" {
  targets = [
    {
      "__path__"    = "/opt/logs/app.log",
      "job"         = "linux-app",
      "environment" = "lab",
    },
  ]

  forward_to = [loki.write.local.receiver]
}

loki.write "local" {
  endpoint {
    url = "http://<LOKI_HOST>:3100/loki/api/v1/push"
  }
}
```

Replace:

```text
<LOKI_HOST>
```

with the Loki hostname or IP address.

## 3. Check Permissions

```bash
ls -l /opt/logs/app.log
```

The Alloy process must be able to read the file.

Do not make logs world-readable just to solve a permission problem.

## 4. Start Alloy

For a systemd installation:

```bash
sudo systemctl restart alloy
sudo systemctl status alloy
```

## 5. Generate More Logs

```bash
echo "$(date) INFO request completed" | sudo tee -a /opt/logs/app.log
echo "$(date) ERROR payment service unavailable" | sudo tee -a /opt/logs/app.log
```

## 6. Query in Grafana

```logql
{job="linux-app"}
```

Search errors:

```logql
{job="linux-app"} |= "ERROR"
```

## What You Learned

```text
File
 ↓
loki.source.file
 ↓
loki.write
 ↓
Loki
 ↓
LogQL
```
