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
