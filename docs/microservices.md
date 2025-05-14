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
