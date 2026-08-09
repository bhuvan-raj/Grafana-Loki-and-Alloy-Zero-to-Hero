# 3. Loki Architecture

Loki is a distributed system whose components can run together or separately.

For learning, start with the **monolithic** architecture.

## Basic Architecture

```text
                ┌───────────────┐
                │    Alloy      │
                └───────┬───────┘
                        │
                        │ Push logs
                        ▼
                ┌───────────────┐
                │     Loki      │
                │               │
                │ Distributor   │
                │ Ingester      │
                │ Querier       │
                │ Query Frontend│
                │ Compactor     │
                └───────┬───────┘
                        │
                        ▼
                 Object Storage
```

## Write Path

```text
Alloy
  ↓
Distributor
  ↓
Ingester
  ↓
Storage
```

### Distributor

Receives incoming log data and distributes it to ingesters.

### Ingester

Handles incoming log streams and builds/stores chunks before persistence.

## Read Path

```text
Grafana
  ↓
Query Frontend
  ↓
Querier
  ↓
Storage
```

### Query Frontend

Can split and coordinate queries.

### Querier

Reads the required data and executes queries.

## Backend

Other components handle tasks such as:

- Compaction
- Index management
- Rules
- Storage/query support

## Mental Model

```text
WRITE
Alloy → Distributor → Ingester → Storage

READ
Grafana → Query Frontend → Querier → Storage
```
