# Microservices

## Overview
The grocery e-commerce platform uses a microservices architecture to ensure scalability for 100,000 daily active users (DAU), 300,000 peak users, and 50,000 SKUs, with high performance (≤500ms search, ≤1s checkout, ≤100ms inventory updates, 99.9% uptime). Each service is independently deployable on Kubernetes/Docker (on-premises, future cloud migration), using Python (Django) for transactional logic or Node.js (NestJS) for real-time/event-driven tasks. Services support grocery-specific needs like real-time inventory, same-day delivery, and substitutions.

## Microservices
The following services handle distinct business domains, optimized for grocery retail:

| Service            | Tech Stack       | Database      | Responsibilities                                                                 |
|--------------------|------------------|---------------|---------------------------------------------------------------------------------|
| Product Catalog    | Django           | PostgreSQL    | Manages 50,000 SKUs, multilingual (English/Vietnamese) descriptions, categories. |
| Inventory          | NestJS           | Firestore     | Real-time stock updates (≤100ms), perishable tracking, substitution rules.      |
| Order              | Django           | PostgreSQL    | Cart management, checkout (≤1s), order history, status tracking.                |
| Payment            | NestJS           | PostgreSQL    | Processes Stripe payments, extensible for MoMo, handles refunds.                |
| User               | Django           | PostgreSQL    | JWT-based authentication, user profiles, role-based access (customer, admin).   |
| Delivery           | NestJS           | Firestore     | Grab/ViettelPost integration, same-day delivery slots (≤500ms), in-store pickup. |
| Promotions         | Django           | PostgreSQL    | Discounts, coupons, perishable-specific promotions (e.g., near-expiry deals).   |
| Notification       | NestJS           | Firestore     | Sends multilingual order/delivery updates via email, SMS, or push.              |
| Search             | NestJS           | Elasticsearch | Fast product search (≤500ms) with filters (e.g., category, perishable).         |
| Recommendation     | Django           | PostgreSQL    | Personalized suggestions (e.g., frequent buys), deferred for post-MVP.          |

### Key Notes
- **Django Services**: Handle transactional, ACID-compliant operations (e.g., Order, User) using PostgreSQL for reliability.
- **NestJS Services**: Optimize for real-time, high-throughput tasks (e.g., Inventory, Delivery) with Firestore for scalability.
- **Grocery Focus**: Inventory tracks perishables (e.g., expiration dates), Delivery supports same-day slots, and Search/Promotions enhance UX for 50,000 SKUs.

## Communication
- **REST**: Synchronous calls for immediate responses (e.g., Order Service checks Inventory stock via `/stock/{sku}`).
- **Kafka**: Asynchronous events for decoupled workflows (e.g., `OrderPlaced` triggers `StockReserved`). Topics include `order_events`, `payment_events`, `delivery_events`.
- **Eventual Consistency**: Use sagas to manage distributed transactions (e.g., rollback inventory if payment fails via compensating events).
- **Service Discovery**: Kubernetes DNS resolves service endpoints (e.g., `inventory-service:8080`).

### Example Kafka Event
```json
{
  "event": "OrderPlaced",
  "orderId": "ord-12345",
  "userId": "usr-67890",
  "items": [
    {"sku": "apple", "quantity": 2, "substitutionAllowed": true}
  ],
  "timestamp": "2025-05-14T13:00:00Z"
}
```

## Scalability and Performance
- **Scaling**: Each service scales independently via Kubernetes Horizontal Pod Autoscaling (HPA) based on CPU (80%) or request rate. Critical services (Inventory, Delivery) prioritize pods during 300,000-user peaks.
- **Caching**: Redis caches frequent data (e.g., product details, stock levels) with TTLs (1–10 minutes) to reduce database load.
- **Performance**: REST calls optimized for ≤500ms (Search), Kafka events for ≤100ms (Inventory updates), and GraphQL for efficient client queries (≤1s checkout).

## Best Practices
- **Modularity**: Services are loosely coupled, with clear APIs (REST/GraphQL) and event schemas (Kafka).
- **Testing**: Unit tests (pytest, Jest), integration tests (Testcontainers), and E2E tests (Cypress) per service, run via GitHub Actions.
- **Monitoring**: Log service interactions to Loki, track metrics (e.g., latency, errors) in Prometheus/Grafana, with alerts for failures.
- **Cloud Prep**: Use cloud-agnostic configs (e.g., environment variables) for future migration to AWS EKS/GCP GKE.
