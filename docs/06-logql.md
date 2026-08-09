# 6. LogQL

**LogQL** is Loki's query language.

## Basic Query

```logql
{job="nginx"}
```

This selects streams where:

```text
job = nginx
```

## Multiple Labels

```logql
{job="nginx", environment="production"}
```

## Search Text

```logql
{job="nginx"} |= "error"
```

## Exclude Text

```logql
{job="nginx"} != "health"
```

## Regular Expression

```logql
{job="nginx"} |~ "error|warning"
```

## JSON Logs

Example:

```json
{"level":"error","service":"payment","message":"database unavailable"}
```

Query:

```logql
{service="payment"} | json
```

Then filter:

```logql
{service="payment"} | json | level="error"
```

## Count Logs

```logql
count_over_time({job="app"}[5m])
```

## Log Rate

```logql
rate({job="app"}[5m])
```

## Query Strategy

Start broad:

```logql
{job="app"}
```

Then narrow:

```logql
{job="app"} |= "error"
```

Then parse structured data if necessary:

```logql
{job="app"} | json | level="error"
```

**Use labels to select streams first, then filter log content.**
