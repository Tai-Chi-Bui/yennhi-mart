# Disaster Recovery

## Backups
- **PostgreSQL**: Incremental (daily, hourly WAL), full (weekly) to MinIO, 30-day retention.
- **Firestore**: Daily exports to MinIO, 14-day retention.
- **Kafka**: Replication factor of 3, 30-day topic backups.
- **Docker**: Images in Harbor, manifests in GitHub.

## Failover
- **Kubernetes**: 3 master nodes, etcd clustering, 3–5 worker nodes.
- **PostgreSQL**: Hot standby replicas with `patroni` for auto-failover.
- **Firestore**: Multi-region replication (hybrid setup).
- **Application**: Circuit breakers (NestJS), anti-affinity for pod distribution.

## Recovery Objectives
| Service | RTO | RPO |
|---------|-----|-----|
| Inventory, Payment, Delivery | ≤15m | ≤5m |
| Others (Recommendation) | ≤1h | ≤1h |

## Best Practices
- Quarterly DR drills (e.g., simulate node failure).
- Monitor failover events in Prometheus/Grafana, alert via Notification Service.
- Document DR playbook in GitHub.
- Cloud Prep: Use S3-compatible MinIO, test multi-region setups.
