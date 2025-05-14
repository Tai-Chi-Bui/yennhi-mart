# Security

## Authentication
- **JWT**: Stateless auth with Django `djangorestframework-simplejwt` and NestJS `@nestjs/jwt`.
- **Future**: OAuth2 for social logins (e.g., Google, Facebook).

## Authorization
- **RBAC**: Roles (customer, admin, warehouse_staff) enforced via Django permissions and NestJS guards.
- Example: Only warehouse staff update inventory.

## Encryption
- **In-Transit**: TLS 1.3 via Cloudflare/Kubernetes Ingress.
- **At-Rest**: PostgreSQL `pgcrypto`, Firestore native encryption.
- **Payment Data**: Tokenized via Stripe.js, only store transaction IDs.

## Payment Security
- **Stripe**: PCI DSS compliance (SAQ-A) using Stripe.js/Elements.
- **Future (MoMo)**: Client-side SDK, HMAC-SHA256 signatures.
- Webhooks validated with signatures, processed via Kafka.

## Additional Measures
- **Rate Limiting**: NGINX Ingress, Cloudflare for API protection.
- **Input Validation**: Django forms, `express-validator` to prevent SQL injection, XSS.
- **Secrets**: API keys in HashiCorp Vault, injected via Kubernetes secrets.
- **Monitoring**: Log security events to Firestore via Kafka, alert via Prometheus/Grafana.
- **Audits**: Plan periodic penetration testing.
