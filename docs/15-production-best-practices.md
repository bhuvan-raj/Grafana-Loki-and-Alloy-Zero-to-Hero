# 15. Production Best Practices

## 1. Keep Labels Low Cardinality

Good:

```text
environment
cluster
namespace
application
job
```

Avoid high-cardinality labels such as:

```text
request_id
trace_id
user_id
IP address
```

unless you have a specific reason and understand the consequences.

## 2. Use Structured Metadata

Use structured metadata for useful information that should not become a high-cardinality stream label.

## 3. Use Object Storage

For production Loki, object storage is an important part of the storage architecture.

Examples:

- Amazon S3
- Google Cloud Storage
- Azure Blob Storage

## 4. Separate Collection from Storage

```text
Applications
     ↓
Alloy
     ↓
Loki
     ↓
Object Storage
```

## 5. Secure Loki

Use:

- Private networking
- TLS where appropriate
- Authentication
- Network policies/security groups
- Least-privilege access

Do not expose Loki write/query endpoints publicly without proper controls.

## 6. Monitor Alloy and Loki

Monitor:

- CPU
- Memory
- Disk
- Ingestion rate
- Query latency
- Errors
- Dropped logs
- Component health

## 7. Avoid Unnecessary Parsing

Do not parse every field if you do not need it.

## 8. Control Log Volume

Consider:

- Dropping unnecessary logs
- Filtering noisy endpoints
- Sampling where appropriate
- Retention policies
- Application log levels

## 9. Start Simple

For learning:

```text
Alloy → Loki → Grafana
```

For larger production environments:

```text
Alloy
  ↓
Gateway / Load Balancer
  ↓
Distributed Loki
  ↓
Object Storage
  ↓
Grafana
```

## 10. Choose Architecture Based on Need

Consider:

```text
Log volume
Query volume
Availability
Retention
Operational requirements
Cost
```
