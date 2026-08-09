# 5. Labels, Streams and Structured Metadata

This is one of the most important Loki topics.

## Labels

A label is a key-value pair:

```text
environment="production"
```

Example:

```text
{job="nginx", environment="production"}
```

## Log Stream

A log stream is identified by its complete set of labels.

These are different streams:

```text
{job="nginx", environment="production"}
{job="nginx", environment="staging"}
```

## Low Cardinality

Use labels for relatively low-cardinality dimensions.

Good examples:

```text
environment
cluster
namespace
application
job
```

Avoid using highly unique values as labels:

```text
request_id
user_id
trace_id
IP address
```

High-cardinality labels can create too many streams.

## Structured Metadata

Structured metadata attaches useful metadata without making it part of the indexed stream labels.

It is useful for high-cardinality information such as:

```text
pod
container_id
trace_id
user_id
```

when those values should not create separate streams.

## Simple Rule

```text
Labels
→ identify the log stream

Structured metadata
→ additional high-cardinality information
```

**Do not turn every field in a log into a Loki label.**
