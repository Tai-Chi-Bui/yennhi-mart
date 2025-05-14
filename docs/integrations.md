# Third-Party Integrations

## Current
- **Stripe**: Payments, PCI DSS compliant (SAQ-A) using Stripe.js/Elements.
- **Grab/ViettelPost**: Delivery, same-day and longer options, in-store pickup.

## Future
- **MoMo**: Local payment gateway for Vietnamese users (QR codes, VND).
- **Gojek, Lalamove**: Additional delivery providers for capacity.

## Technical Approach
- **Abstraction**: Strategy pattern in Payment/Delivery Services (NestJS).
  - Interfaces: `PaymentProcessor` (processPayment), `DeliveryProvider` (scheduleDelivery).
- **REST**: Sync API calls (e.g., MoMo payments) with NestJS `@nestjs/axios`.
- **Kafka**: Async webhooks (e.g., payment confirmation) on topics like `payment_events`.
- **Caching**: Redis for static data (e.g., MoMo bank lists, Gojek rates), TTL 1h.
- **Security**:
  - API keys in HashiCorp Vault, webhook signature validation.
  - TLS 1.3, client-side SDKs for PCI DSS (MoMo).
- **Scalability**:
  - Kubernetes HPA for Payment/Delivery Services.
  - Partition Kafka topics by provider/region.
  - Shard PostgreSQL/Firestore for transaction logs.

## Testing
- Mock APIs with Testcontainers in GitHub Actions.
- E2E tests (Cypress) for checkout, slot booking.
- Load tests (Locust) for 300,000-user performance.

## Monitoring
- Log errors to Loki via Kafka, track latency in Prometheus/Grafana.
- Alerts for failures (e.g., MoMo errors >1%).
