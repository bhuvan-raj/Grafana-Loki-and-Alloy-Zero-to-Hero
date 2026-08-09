# 9. Alloy Architecture and Components

Alloy uses a **component-based pipeline**.

Each component performs a specific task.

## Basic Pipeline

```text
SOURCE
  ↓
PROCESS
  ↓
WRITE
```

For logs:

```text
Log File
   ↓
loki.source.file
   ↓
loki.process
   ↓
loki.write
   ↓
Loki
```

## Source Components

Examples:

```text
loki.source.file
loki.source.docker
loki.source.kubernetes
loki.source.journal
```

## Process Component

The main processing component is:

```text
loki.process
```

It can:

- Parse logs
- Extract fields
- Add labels
- Drop logs
- Rewrite labels
- Filter logs

## Write Component

```text
loki.write
```

sends logs to a Loki endpoint.

## Example

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

## Kubernetes

Alloy also provides Kubernetes-specific log sources such as:

```text
loki.source.kubernetes
```

which can tail Kubernetes Pod logs through the Kubernetes API.

## Mental Model

```text
Collect → Transform → Send
```
