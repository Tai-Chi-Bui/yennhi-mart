# System Architecture

## Overview
The platform uses a microservices architecture to ensure scalability, performance, and maintainability. It runs on Kubernetes/Docker on-premises, with plans for cloud migration (e.g., AWS EKS). The system supports 100,000 DAU, 300,000 peak users, and 50,000 SKUs, with grocery-specific features (e.g., real-time inventory, delivery scheduling).

## Tech Stack
- **Frontend**: React/Vue.js (planned), served via GraphQL/REST.
- **Backend**: Python (Django) for transactional services, Node.js (NestJS) for real-time/event-driven services.
- **Databases**:
  - PostgreSQL: Transactional data (orders, users, catalog).
  - Firestore: Real-time data (inventory, delivery slots).
- **Messaging**: Kafka for asynchronous events (e.g., order placement, stock updates).
- **Caching**: Redis for hot data (e.g., product details, sessions).
- **CDN**: Cloudflare/CloudFront for static assets and cached responses.
- **Orchestration**: Kubernetes/Docker with NGINX Ingress.
- **CI/CD**: GitHub Actions with self-hosted runners.
- **Monitoring**: Prometheus/Grafana, Loki, OpenTelemetry/Jaeger.
- **Third-Party**: Stripe (payments), Grab/ViettelPost (delivery), extensible for MoMo, Gojek.

## Architecture Diagram
(TODO: Add diagram using tools like Draw.io or Excalidraw)
- Frontend ↔ GraphQL/REST APIs ↔ Microservices (Kubernetes pods).
- Microservices communicate via REST (sync) and Kafka (async).
- PostgreSQL/Firestore for data persistence, Redis for caching.
- External APIs (Stripe, Grab) integrated via REST and Kafka webhooks.

## Communication
- **REST**: Synchronous calls (e.g., User auth, Payment processing).
- **GraphQL**: Client-facing queries (e.g., product listings, user profiles).
- **Kafka**: Asynchronous events (e.g., `OrderPlaced` → `StockReserved`). Sagas for eventual consistency.
- **Service Discovery**: Kubernetes DNS for internal service routing.

## Scalability
- Kubernetes HPA and Cluster Autoscaler for pod/node scaling.
- PostgreSQL read replicas, sharding (future), Firestore auto-scaling.
- Redis caching and Cloudflare CDN for low latency.

## Security
- JWT for authentication, RBAC for authorization.
- TLS 1.3 (Cloudflare/Ingress), at-rest encryption (PostgreSQL `pgcrypto`, Firestore).
- API keys in HashiCorp Vault, webhook signature validation.
