# Performance

## Goals
| Action | Target | Notes |
|--------|--------|-------|
| Search | ≤500ms | Elasticsearch/Firestore, Redis caching. |
| Page Load | ≤2s | Cloudflare CDN, optimized frontend. |
| Checkout | ≤1s/step | Stripe integration, Order Service. |
| Inventory Update | ≤100ms | Firestore, Kafka events. |
| Delivery Scheduling | ≤500ms | Firestore, Redis-cached slots. |
| Uptime | ≥99.9% | Kubernetes HA, failover mechanisms. |

## Strategies
- **Caching**: Redis for product data, sessions, stock (TTLs: 1–10 min).
- **CDN**: Cloudflare/CloudFront for static assets and cached APIs.
- **Database Optimization**:
  - PostgreSQL: Indexes (GIN, BRIN), read replicas.
  - Firestore: Composite indexes, batch writes.
- **Load Balancing**: Kubernetes Ingress, future cloud load balancers.
- **Auto-Scaling**: Kubernetes HPA for pods, Cluster Autoscaler for nodes.
- **Testing**: Locust for 300,000-user loads, Lighthouse for page performance.

## Monitoring
- Prometheus/Grafana for latency, error rates, and uptime.
- Alerts for thresholds (e.g., search >500ms, uptime <99.9%).
