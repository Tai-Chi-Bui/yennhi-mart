# Grocery E-Commerce Platform

## Overview
A large-scale, microservices-based e-commerce platform for a grocery store, designed to support 100,000 daily active users (DAU), 300,000 peak users, and 50,000 SKUs. The platform prioritizes scalability, performance, and grocery-specific features, with an on-premises Kubernetes/Docker setup and plans for future cloud migration.

## Key Requirements
- **Scale**: 100,000 DAU, 300,000 peak users (3x during sales), 50,000 SKUs.
- **Features**: Product browsing, search, cart, checkout, payments, order tracking, user accounts, promotions, inventory management, delivery scheduling, in-store pickup, perishable goods handling, and substitutions.
- **Localization**: English and Vietnamese support, no specific regional compliance yet.
- **Delivery**: Grab and ViettelPost, with same-day and longer delivery options.
- **Priorities**: Scalability and performance (≤500ms search, ≤2s page load, ≤100ms inventory updates, 99.9% uptime).

## Tech Stack
- **Languages/Frameworks**: Python (Django), Node.js (NestJS).
- **Databases**: PostgreSQL (transactional), Firestore (real-time).
- **Infrastructure**: Kubernetes/Docker (on-premises), future cloud migration (AWS/GCP/Azure).
- **APIs/Communication**: REST (sync), GraphQL (client), Kafka (async).
- **Third-Party**: Stripe (payments), Grab/ViettelPost (delivery), extensible for MoMo, Gojek.

## Documentation
- [Architecture](docs/architecture.md): System design and microservices.
- [Microservices](docs/microservices.md): Service breakdown and responsibilities.
- [Scalability](docs/scalability.md): Strategies for high traffic and growth.
- [Performance](docs/performance.md): Response time and uptime goals.
- [Security](docs/security.md): Authentication, encryption, and compliance.
- [Deployment](docs/deployment.md): CI/CD and deployment strategies.
- [Monitoring](docs/monitoring.md): Logging, metrics, and observability.
- [Testing](docs/testing.md): Unit, integration, E2E, and load testing.
- [Technical Debt](docs/technical-debt.md): Managing and prioritizing debt.
- [Disaster Recovery](docs/disaster-recovery.md): Backups and failover.
- [UX Validation](docs/ux-validation.md): Prototyping, feedback, and A/B testing.
- [Success Metrics](docs/success-metrics.md): KPIs and analytics.
- [Third-Party Integrations](docs/integrations.md): Stripe, delivery providers, and future partners.

## Getting Started
1. Clone the repo: `git clone <repo-url>`.
2. Set up Kubernetes/Docker on-premises.
3. Install dependencies: `pip install -r requirements.txt` (Django), `npm install` (NestJS).
4. Configure databases (PostgreSQL, Firestore) and third-party APIs (Stripe, Grab/ViettelPost).
5. Deploy using GitHub Actions CI/CD (see [Deployment](docs/deployment.md)).

## Contact
Solo developer: Manage issues and tasks via GitHub Issues and Projects.
