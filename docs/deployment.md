# Deployment

## CI/CD
- **Tool**: GitHub Actions with self-hosted runners (on-premises).
- **Pipeline**:
  - **Build**: Compile Django/NestJS, lint (Flake8, ESLint), build Docker images.
  - **Test**: Unit (pytest, Jest), integration (Testcontainers), E2E (Cypress), load (Locust).
  - **Deploy**: Push images to Harbor, apply Kubernetes manifests via `kubectl`.
- **Secrets**: Stored in HashiCorp Vault, injected as Kubernetes secrets.

## Deployment Types
- **Blue-Green**: For Inventory, Delivery, Payment to ensure zero-downtime.
- **Canary**: For high-risk changes (e.g., new MoMo integration).
- **Rolling Updates**: For non-critical services (e.g., Recommendation).
- **Automation**: Helm charts for Kubernetes, automated rollbacks via GitHub Actions.

## Best Practices
- **Zero-Downtime**: Liveness/readiness probes, `maxUnavailable=0`.
- **Environment Parity**: Staging mirrors production (Kubernetes, Firestore emulator).
- **Post-Deployment**: Smoke tests (Postman) for critical flows (checkout, slots).
- **Cloud Prep**: Cloud-agnostic manifests for AWS EKS/GCP GKE.

## Example GitHub Actions Workflow
```yaml
name: Deploy Inventory Service
on: push
jobs:
  build-test-deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker
        run: docker build -t harbor.local/inventory:latest .
      - name: Test
        run: npm test
      - name: Deploy
        run: kubectl apply -f k8s/inventory.yaml



---

#### File 8: `docs/monitoring.md`
```markdown
# Monitoring and Observability

## Tools
- **Loki + Grafana**: Log aggregation (Django/NestJS, Kafka) with Promtail sidecars.
- **Prometheus + Grafana**: Metrics and alerting (latency, errors, uptime).
- **OpenTelemetry + Jaeger**: Distributed tracing for latency debugging.

## Key Metrics
- **Performance**: API latency (≤500ms search, ≤1s checkout), inventory updates (≤100ms), page loads (≤2s).
- **System**: Kubernetes CPU/memory, Redis hit/miss, PostgreSQL/Firestore performance.
- **Grocery-Specific**: Inventory accuracy (≥99%), slot booking success (≥98%), substitution acceptance (≥70%).
- **Errors**: HTTP 5xx, Stripe failures, Kafka lag.

## Implementation
- Log events to Firestore via Kafka (e.g., order failures).
- Export metrics to Prometheus (e.g., `orders_completed_total`).
- Visualize in Grafana dashboards, alert via Notification Service.
- Retention: Logs (30 days), critical events (90 days).

## Example Prometheus Metric
```promql
http_request_duration_seconds{service="inventory", endpoint="/stock"}[5m]


---

#### File 9: `docs/testing.md`
```markdown
# Testing

## Types
| Type | Tools | Focus |
|------|-------|-------|
| Unit | pytest, Jest | Django views, NestJS controllers (≥80% coverage). |
| Integration | pytest, supertest, Testcontainers | REST/Kafka interactions, Stripe/Grab APIs. |
| E2E | Cypress | User flows (checkout, slot booking, multilingual UX). |
| Load | Locust | 300,000-user performance (≤500ms search). |
| Security | OWASP ZAP, Snyk | Vulnerabilities, dependency scans. |

## Workflow
- **PR**: Unit tests, linting, Snyk scans in GitHub Actions.
- **Build**: Integration tests with Testcontainers.
- **Pre-Deploy**: E2E tests in staging (Kubernetes, Firestore emulator).
- **Periodic**: Load tests weekly, security scans monthly.
- **Data**: Use `factory_boy` for realistic test data (e.g., 50,000 SKUs).

## Example Test
```python
# tests/test_inventory.py
import pytest
from inventory.models import Stock

@pytest.mark.django_db
def test_stock_update():
    stock = Stock.objects.create(sku="apple", quantity=10)
    stock.decrement(2)
    assert stock.quantity == 8


---

#### File 10: `docs/technical-debt.md`
```markdown
# Technical Debt

## Identification
- **Code Reviews**: Use CodeClimate/SonarQube in GitHub PRs to flag code smells.
- **Static Analysis**: Flake8, ESLint, Snyk for vulnerabilities and inefficiencies.
- **Monitoring**: Prometheus/Grafana for performance issues (e.g., slow queries >500ms).
- **Audits**: Quarterly reviews, logged in GitHub Issues.

## Addressing
- **Refactoring**: 10–20% sprint time (e.g., optimize Firestore queries).
- **Incremental**: Fix debt during feature work (e.g., refactor Django model).
- **Backlog**: Prioritize by impact (e.g., high for inventory latency, low for UI).
- **Automation**: Fail builds if coverage <80% or Snyk flags critical issues.

## Balancing
- **80/20 Rule**: 80% new features, 20% debt/maintenance.
- **Performance Budgets**: Reject features exceeding latency goals (e.g., >500ms search).
- **Feature Flags**: Use Django `waffle` for safe rollouts, reducing debt risk.

## Grocery-Specific
- Prioritize debt in Inventory (e.g., race conditions) and Delivery (e.g., slot logic) to ensure ≤100ms updates and ≤500ms slot availability.
