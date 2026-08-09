# 11. Collecting and Processing Logs with Alloy

This chapter builds the core Alloy logging pipeline.

## 1. Collect a File

Use:

```text
loki.source.file
```

Example:

```alloy
loki.source.file "app" {
  targets = [
    {
      "__path__" = "/tmp/app.log",
      "job"     = "app",
    },
  ]

  forward_to = [loki.write.local.receiver]
}
```

## 2. Discover Files

The current `loki.source.file` component has a built-in `file_match` block.

Example:

```alloy
loki.source.file "logs" {
  targets = [
    {
      "__path__" = "/var/log/*.log",
      "job"     = "system",
    },
  ]

  file_match {
    enabled = true
  }

  forward_to = [loki.write.local.receiver]
}
```

Use `local.file_match` when you specifically need to share discovered targets between components or use another discovery/relabeling step.

## 3. Process Logs

Example:

```alloy
loki.process "app" {
  stage.json {
    expressions = {
      level   = "level",
      service = "service",
    }
  }

  stage.labels {
    values = {
      level   = "",
      service = "",
    }
  }

  forward_to = [loki.write.local.receiver]
}
```

The source can forward to the process component:

```alloy
forward_to = [loki.process.app.receiver]
```

## 4. Send to Loki

```alloy
loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

## Complete Pipeline

```text
/var/log/app.log
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

## Common Processing Tasks

- Parse JSON
- Extract fields
- Add labels
- Drop logs
- Rewrite labels
- Filter logs

Do not automatically convert every parsed field into a Loki label.
