# 10. Alloy Configuration

Alloy uses its own configuration language.

Configuration files conventionally use:

```text
.alloy
```

## Blocks

Example:

```alloy
loki.write "local" {
  endpoint {
    url = "http://localhost:3100/loki/api/v1/push"
  }
}
```

Here:

```text
loki.write
```

is the component type and:

```text
local
```

is its label.

## Attributes

Example:

```alloy
url = "http://localhost:3100/loki/api/v1/push"
```

## Component References

Example:

```alloy
forward_to = [loki.write.local.receiver]
```

This connects one component to another.

## Simple Pipeline

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
    url = "http://localhost:3100/loki/api/v1/push"
  }
}
```

## Formatting

Use:

```bash
alloy fmt config.alloy
```

## Mental Model

```text
Block
 ├── Attributes
 └── Nested blocks
```

and:

```text
Component A
     │
     │ receiver
     ▼
Component B
```

Alloy determines dependencies from these component references.
