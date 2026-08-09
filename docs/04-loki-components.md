# 4. Loki Components

You do not need to memorize every Loki component initially. Understand the main write and read paths first.

## Distributor

Receives logs from clients such as Alloy.

```text
Alloy → Distributor
```

## Ingester

Handles log data received from distributors and builds chunks.

```text
Distributor → Ingester
```

## Querier

Executes log queries against Loki data.

```text
Grafana → Querier
```

## Query Frontend

Coordinates queries and can split them into smaller requests.

```text
Grafana
   ↓
Query Frontend
   ↓
Querier
```

## Compactor

Performs background storage/index maintenance tasks.

## Index Gateway

Handles access to index data in distributed deployments.

## Ruler

Evaluates recording and alerting rules.

## Gateway

Provides an entry point in front of Loki components and can route requests appropriately.

## Remember

```text
Distributor → receives logs
Ingester    → handles incoming streams
Querier     → executes queries
Frontend    → coordinates queries
Compactor   → maintains storage/index data
Ruler       → evaluates rules
Gateway     → provides an entry point
```
