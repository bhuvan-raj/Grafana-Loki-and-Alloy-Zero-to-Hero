# 7. Loki Storage and Deployment Modes

## Storage

Modern Loki deployments commonly use object storage for chunks and index data.

Examples:

- Amazon S3
- Google Cloud Storage
- Azure Blob Storage

Object storage provides durable, scalable storage for Loki data.

## Deployment Modes

### Monolithic

All major Loki components run in one process.

```text
Loki
└── All components
```

Useful for:

- Learning
- Development
- Small environments
- Meta-monitoring

### Microservices / Distributed

Loki components run as separate services.

```text
Distributor
Ingester
Querier
Query Frontend
Compactor
...
```

This allows components to scale independently and is appropriate for high-scale production environments.

### Simple Scalable Deployment

Simple Scalable Deployment separates Loki into read, write, and backend paths.

However, **SSD is being deprecated**, so it should not be the primary new production architecture taught here.

## Recommended Learning Path

```text
Monolithic
   ↓
Understand components
   ↓
Understand storage
   ↓
Understand distributed mode
```

Choose a deployment mode based on:

- Log volume
- Query volume
- Availability
- Retention
- Storage backend
- Operational complexity
