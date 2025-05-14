Deployment
Overview
The grocery e-commerce platform is deployed on Kubernetes/Docker (on-premises, with future cloud migration to AWS EKS/GCP GKE) to support 100,000 daily active users (DAU), 300,000 peak users, and 50,000 SKUs. Deployments prioritize zero-downtime, performance (≤500ms search, ≤1s checkout, ≤100ms inventory updates), and reliability (99.9% uptime), ensuring uninterrupted grocery operations (e.g., real-time inventory, same-day delivery).
CI/CD

Tool: GitHub Actions with self-hosted runners on Kubernetes, minimizing costs and aligning with on-premises setup.
Pipeline:
Build: Compile Django/NestJS code, run linters (Flake8, ESLint), and build Docker images for microservices (e.g., Inventory, Delivery).
Test: Execute unit tests (pytest, Jest), integration tests (Testcontainers for PostgreSQL/Firestore), E2E tests (Cypress for checkout flows), and load tests (Locust for 300,000 users).
Deploy: Push images to Harbor (private registry), apply Kubernetes manifests via kubectl, and run smoke tests (e.g., checkout, slot booking).


Secrets: Store sensitive data (e.g., Stripe API keys, database credentials) in HashiCorp Vault, injected as Kubernetes secrets during deployment.

Deployment Types

Blue-Green: Used for critical services (Inventory, Delivery, Payment) to ensure zero-downtime, vital for real-time stock updates (≤100ms) and delivery slots (≤500ms). Deploy new version to a separate namespace, validate, then switch traffic via Kubernetes Ingress.
Canary: Applied for high-risk updates (e.g., new MoMo integration), rolling out to 5% of users, monitoring metrics (Prometheus/Grafana), and expanding if stable.
Rolling Updates: Employed for non-critical services (e.g., Recommendation, Promotions) to minimize resource usage while maintaining availability.
Automation: Helm charts define Kubernetes deployments, with GitHub Actions handling rollout and rollback logic.

Best Practices

Zero-Downtime: Configure liveness/readiness probes and set maxUnavailable=0 in Kubernetes manifests to prevent disruptions during updates, critical for grocery checkout (≤1s).
Environment Parity: Maintain a staging environment mirroring production (on-premises Kubernetes, Firestore emulator, PostgreSQL) to catch issues early.
Post-Deployment Validation: Run automated smoke tests (e.g., via Postman in GitHub Actions) to verify critical flows (e.g., order placement, delivery slot booking).
Grocery-Specific:
Prioritize blue-green for Inventory to avoid stock oversells, ensuring ≤100ms updates.
Use canary for Delivery Service changes to validate Grab/ViettelPost integrations without impacting same-day delivery.


Monitoring: Log deployment events to Loki via Kafka, track success/failure in Prometheus/Grafana, and alert via Notification Service for issues (e.g., failed rollout).
Cloud Prep: Use cloud-agnostic manifests (e.g., avoid on-premises-specific configs) and test deployments in staging to ensure compatibility with AWS EKS/GCP GKE.

Example GitHub Actions Workflow
name: Deploy Order Service
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker Image
        run: docker build -t harbor.local/order:latest ./services/order
      - name: Run Tests
        run: cd services/order && poetry run pytest
      - name: Push to Harbor
        run: docker push harbor.local/order:latest
      - name: Deploy to Kubernetes
        run: kubectl apply -f k8s/order-deployment.yaml
      - name: Smoke Test
        run: curl --fail http://order-service/health

Notes

Performance: Blue-green deployments ensure ≤1s checkout and ≤500ms slot updates during updates, critical for 300,000 peak users.
Scalability: Helm charts and HPA enable rapid scaling of services (e.g., Payment during sales).
Security: Secrets management and smoke tests maintain JWT/RBAC integrity and Stripe PCI DSS compliance.

