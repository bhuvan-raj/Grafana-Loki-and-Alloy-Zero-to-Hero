# 14. Troubleshooting

Troubleshoot from left to right:

```text
Source
  ↓
Alloy
  ↓
Network
  ↓
Loki
  ↓
Grafana
```

## 1. Check Alloy

```bash
sudo systemctl status alloy
```

Logs:

```bash
sudo journalctl -u alloy -n 100 --no-pager
```

## 2. Format Alloy Configuration

```bash
alloy fmt config.alloy
```

If Alloy fails to start, inspect the service logs for the exact configuration error.

## 3. Check the Alloy UI

Alloy provides a local UI for inspecting component health and pipelines. The address depends on the configured listener.

## 4. Check Loki

```bash
curl http://localhost:3100/ready
```

## 5. Check Loki Logs

Docker:

```bash
docker logs loki
```

Kubernetes:

```bash
kubectl logs <loki-pod>
```

## 6. Test Network Connectivity

```bash
curl http://loki:3100/ready
```

or:

```bash
curl http://<LOKI_HOST>:3100/ready
```

## 7. Check Grafana

In Grafana:

```text
Connections
→ Data sources
→ Loki
→ Save & test
```

## 8. No Logs in Grafana

Check:

```text
Is the source producing logs?
        ↓
Can Alloy read the logs?
        ↓
Is Alloy forwarding them?
        ↓
Can Alloy reach Loki?
        ↓
Is Loki accepting them?
        ↓
Does the LogQL query select the right labels?
```

## Common Problems

### Permission denied

Check:

```bash
ls -l /path/to/log
```

Make sure the Alloy process can read the file.

### Wrong file path

Check the exact path configured in `loki.source.file`.

### Wrong Loki URL

For Docker Compose, it may be:

```text
http://loki:3100/loki/api/v1/push
```

### Wrong labels

Start broad:

```logql
{}
```

Then narrow:

```logql
{job="app"}
```

## Golden Rule

```text
SOURCE → ALLOY → LOKI → GRAFANA
```
