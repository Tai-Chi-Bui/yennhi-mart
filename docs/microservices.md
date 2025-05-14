# Microservices

## Overview
The platform is divided into independent microservices to ensure scalability and maintainability, each handling a specific business domain. Services use Django (transactional) or NestJS (real-time), with PostgreSQL or Firestore for persistence.

## Services
| Service | Tech Stack | Database | Responsibilities |
|---------|------------|----------|-----------------|
| Product Catalog | Django | PostgreSQL | Manage 50,000 SKUs, search, multilingual descriptions. |
| Inventory | NestJS | Firestore | Real-time stock updates (≤100ms), perishables, substitutions. |
| Order | Django | PostgreSQL | Cart, checkout (≤1s), order tracking. |
| Payment | NestJS | PostgreSQL | Stripe integration, extensible for MoMo. |
| User | Django | PostgreSQL | JWT auth, user profiles, RBAC. |
| Delivery | NestJS | Firestore | Grab/ViettelPost integration, same-day slots (≤500ms). |
| Promotions | Django | PostgreSQL | Discounts, grocery-specific promotions. |
| Notification | NestJS | Firestore | Multilingual order/delivery updates. |
| Search | NestJS | Elasticsearch/Firestore | Fast product search (≤500ms). |
| Recommendation | Django | PostgreSQL/Firestore | Personalized suggestions (post-MVP). |

## Communication
- **REST**: Sync calls (e.g., Order → Inventory for stock check).
- **Kafka**: Async events (e.g., `OrderPlaced` → `StockReserved`). Topics: `payment_events`, `delivery_events`.
- **Eventual Consistency**: Sagas for distributed transactions (e.g., rollback inventory if payment fails).

## Example Kafka Event
```json
// Topic: order_events
{
  "event": "OrderPlaced",
  "orderId": "12345",
  "items": [{"sku": "apple", "quantity": 2}],
  "timestamp": "2025-05-14T13:00:00Z"
}


---

#### File 4: `docs/scalability.md`
```markdown
# Scalability

## Overview
The platform is designed to handle 100,000 DAU and 300,000 peak users, with plans for growth beyond (e.g., millions of users, 100,000+ SKUs, new regions).

## Strategies
- **Horizontal Scaling**:
  - Kubernetes HPA: Scale pods based on CPU/memory (80% threshold) or request rate.
  - Cluster Autoscaler: Add nodes during peaks (e.g., 500,000 users).
  - Split services (e.g., Search Indexing, Order Processing) for load distribution.
- **Database Scaling**:
  - **PostgreSQL**: 2–3 read replicas for reads, partitioning by date/user, sharding (Citus) for millions of orders.
  - **Firestore**: Auto-scaling, optimized collections (e.g., `inventory/{sku}/{region}`), Redis caching for reads.
- **Caching**: Redis for hot data (e.g., stock, slots) with TTLs (1–10 minutes).
- **CDN**: Cloudflare/CloudFront for static assets and cached GraphQL responses.
- **Load Balancing**: Kubernetes Ingress (NGINX), future cloud load balancers (AWS ALB).

## Regional Expansion
- On-premises: Simulate regions with separate clusters (e.g., Hanoi, HCMC) and data replication.
- Cloud: Multi-region deployment (e.g., AWS Singapore, US) with global load balancing (AWS Global Accelerator).
- Multilingual: Extend Django i18n/NestJS `i18next` for new languages (e.g., Thai).

## New Features
- Modular microservices (e.g., Loyalty, Subscription) using NestJS.
- Feature flags (Django `waffle`) for incremental rollout.
- Extensible schemas in PostgreSQL/Firestore (e.g., `loyalty_points` table).

## Monitoring
- Prometheus/Grafana to forecast traffic (e.g., 1M users).
- Locust for capacity testing (e.g., 500,000-user simulation).
