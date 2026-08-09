# 12. Alloy → Loki

The main Alloy logging job is:

```text
Collect
   ↓
Process
   ↓
Send
```

## `loki.write`

```alloy
loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

## Remote Loki

```alloy
loki.write "remote" {
  endpoint {
    url = "https://example.com/loki/api/v1/push"
  }
}
```

Authentication can be configured when required.

## Complete Example

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

loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

## With Processing

```alloy
loki.source.file "app" {
  targets = [
    {
      "__path__" = "/tmp/app.log",
      "job"     = "app",
    },
  ]

  forward_to = [loki.process.app.receiver]
}

loki.process "app" {
  forward_to = [loki.write.local.receiver]
}

loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

The pipeline is:

```text
Source → Process → Write → Loki
```
