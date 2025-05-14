# Development Plan

## Overview
This plan outlines the development of a large-scale, microservices-based grocery e-commerce platform supporting 100,000 daily active users (DAU), 300,000 peak users, and 50,000 SKUs. The platform prioritizes scalability, performance (≤500ms search, ≤2s page load, ≤100ms inventory updates, 99.9% uptime), and grocery-specific features (e.g., real-time inventory, same-day delivery, substitutions, multilingual English/Vietnamese UX). It runs on Kubernetes/Docker on-premises, with future cloud migration (e.g., AWS EKS). As a solo developer, the plan focuses on automation, prioritization, and modularity to streamline implementation.

## Scope
- **MVP Features**:
  - Product Catalog: Browse/search 50,000 SKUs, multilingual descriptions.
  - Inventory: Real-time stock updates (≤100ms), perishable tracking, substitutions.
  - Order: Cart, checkout (≤1s), order tracking.
  - Payment: Stripe integration, PCI DSS compliant.
  - User: JWT-based authentication, basic profiles.
  - Delivery: Grab/ViettelPost integration, same-day slots (≤500ms), in-store pickup.
  - Notifications: Basic order/delivery updates (email/SMS).
- **Post-MVP Features** (Deferred):
  - Promotions: Discounts, perishable-specific deals.
  - Search: Advanced filtering (e.g., Elasticsearch).
  - Recommendation: Personalized suggestions.
  - Additional integrations (e.g., MoMo, Gojek).
- **Non-Functional**:
  - Scalability: Handle 300,000 peak users.
  - Performance: ≤500ms search, ≤2s page load, ≤100ms inventory updates.
  - Uptime: ≥99.9%.
  - Security: JWT, RBAC, TLS 1.3, PCI DSS compliance.

## Tech Stack
- **Backend**: Python (Django) for transactional services, Node.js (NestJS) for real-time services.
- **Databases**: PostgreSQL (transactional), Firestore (real-time).
- **Infrastructure**: Kubernetes/Docker, NGINX Ingress, Redis (caching), Cloudflare (CDN).
- **Messaging**: Kafka for async events.
- **CI/CD**: GitHub Actions with self-hosted runners.
- **Monitoring**: Prometheus/Grafana, Loki, OpenTelemetry/Jaeger.
- **Testing**: pytest, Jest, Cypress, Locust, Testcontainers.
- **Third-Party**: Stripe (payments), Grab/ViettelPost (delivery).

## Development Phases
The plan is divided into phases, with estimated durations assuming 20–30 hours/week of solo development. Timelines are approximate and can be adjusted based on progress.

### Phase 1: Project Setup and Infrastructure (1–2 Weeks)
**Goal**: Establish a lightweight development environment using Docker on a work laptop (~8–12GB RAM, 4 cores), focusing on core services (PostgreSQL, Firestore emulator, Redis) and a minimal Kubernetes cluster (`kind`) to support grocery-specific features (e.g., real-time inventory, delivery slots). Defer non-essential services (Kafka, monitoring, Harbor) to later phases to reduce resource usage.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 1.1 Set up repo | Initialize GitHub repo, create monorepo structure (`/services`, `/docs`). | 1 day | None | Use GitHub CLI: `gh repo create grocery-ecommerce`. Structure: `/services/inventory`, `/services/order`, `/docs`. Create GitHub Project Kanban board with labels (`phase-1`, `grocery`). |
| 1.2 Configure local env | Set up a Docker-based dev container for Python, Node.js, and dependencies. | 1 day | 1.1 | Use VS Code Dev Containers with `mcr.microsoft.com/vscode/devcontainers/javascript-node:20` + Python. Install `poetry`, `npm`. Limit container to 2GB RAM. See `.devcontainer/devcontainer.json`. |
| 1.3 Set up Kubernetes | Deploy a minimal `kind` cluster (1 control-plane, 1 worker) for local testing. Configure NGINX Ingress. | 2 days | 1.2 | Install `kind`: `curl -Lo kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64`. Create cluster: `kind create cluster --name grocery`. Deploy NGINX: `kubectl apply -f ingress-nginx.yaml`. Limit to ~2GB RAM. Defer on-premises cluster to Phase 6. |
| 1.4 Configure databases | Run PostgreSQL and Firestore emulator in Docker. Seed with 100 SKUs for grocery testing. | 2 days | 1.2 | PostgreSQL: `docker run -d postgres:14-alpine` (512MB). Firestore: `docker run -d gcr.io/google.com/cloudsdktool/cloud-sdk:emulators` (512MB). Seed PostgreSQL: `INSERT INTO products (sku, name_en, name_vi, is_perishable) VALUES ('milk', 'Milk', 'Sữa', true);`. Seed Firestore: `{sku: "milk", stock: 50, expires: "2025-05-20"}`. Use `docker-compose.yml`. |
| 1.5 Set up Redis | Run Redis for caching delivery slots. | 1 day | 1.3 | `docker run -d redis:7.0-alpine --maxmemory 512mb` (512MB). Test caching: `SET slots:2025-05-15 "{\"time\": \"10:00\", \"available\": true}"`. Include in `docker-compose.yml`. |
| 1.6 Configure CI/CD | Create GitHub Actions workflow for build/test, using Docker Hub. | 2 days | 1.1 | Workflow: lint (Flake8, ESLint), test (pytest, Jest), build Docker images. Example: `docker build -t grocery/inventory:test`. Defer Harbor to Phase 6. See `.github/workflows/ci.yml`. |
| 1.7 Defer Kafka | Mock event-driven logic (e.g., `OrderPlaced`) in-memory until Phase 2. | 0 days | None | Use Python `asyncio.Queue` or Node.js `EventEmitter` for local testing. Deploy Kafka (`bitnami/kafka`) in Phase 2 for Inventory service. |
| 1.8 Defer monitoring | Use Docker logs and stats initially. Deploy Prometheus/Grafana in Phase 5. | 0 days | None | Run `docker stats` for resource monitoring. Add Prometheus (`prom/prometheus`), Grafana (`grafana/grafana`) in Phase 5 for performance testing. |

**Grocery Focus**:
- **Inventory**: Seed Firestore with perishable SKUs to test ≤100ms updates (e.g., stock changes).
- **Delivery**: Use Redis to cache mock slots (≤500ms) for same-day delivery testing.
- **Validation**: Test database and cache performance on laptop to ensure grocery features work locally.

**Resource Considerations**:
- **Laptop Specs**: Assume ~8–12GB RAM, 4 cores. Limit containers to ~6GB total (512MB each for PostgreSQL, Firestore, Redis; 2GB for dev container; 2GB for `kind`).
- **Monitoring**: Run `docker stats` to track CPU/RAM. Stop unused containers (`docker stop <name>`).
- **Constraints**: Check work laptop’s IT policies for Docker/network restrictions. Use WSL2 (Windows) or Linux VM if needed.

**Docker Compose Setup**:
```yaml
version: '3'
services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    deploy:
      resources:
        limits:
          memory: 512m
  firestore-emulator:
    image: gcr.io/google.com/cloudsdktool/cloud-sdk:emulators
    command: gcloud beta emulators firestore start --host-port=0.0.0.0:8080
    ports:
      - "8080:8080"
    deploy:
      resources:
        limits:
          memory: 512m
  redis:
    image: redis:7.0-alpine
    command: redis-server --maxmemory 512mb
    ports:
      - "6379:6379"
    deploy:
      resources:
        limits:
          memory: 512m
```
Run: `docker-compose up -d`.

**Output**: GitHub repo with `/docs`, `/services`, lightweight Docker environment (PostgreSQL, Firestore emulator, Redis), minimal `kind` cluster, and CI/CD pipeline. Ready for grocery service development in Phase 2.

### Phase 2: Core Microservices Development (8–10 Weeks)
**Goal**: Build MVP microservices (Product Catalog, Inventory, Order, Payment, User, Delivery, Notification) with grocery-specific functionality, leveraging Phase 1’s Docker environment (PostgreSQL, Firestore emulator, Redis, `kind` cluster). Each service includes detailed API schemas, database models, Kafka integration (added in Task 2.2), and automated tests to ensure reliability on a work laptop (~8–12GB RAM, 4 cores).

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 2.1 Product Catalog | Build service to manage 50,000 SKUs with multilingual descriptions and basic search. | 1.5 weeks | 1.4 | **Tech**: Django, PostgreSQL (`postgres:14-alpine`). **Models**: `Product(sku: str, name_en: str, name_vi: str, category: str, price: decimal, is_perishable: bool, expires_at: datetime)`. **API**: REST (`/products`, `/products/{sku}`) with GET endpoints. **Search**: PostgreSQL GIN index on `name_en`, `name_vi` for ≤500ms queries. **Config**: Use `django-environ` for env vars (`DATABASE_URL`). **Script**: Seed 10,000 SKUs (`scripts/seed_products.py`) with 20% perishables. **Tests**: pytest for model validation, API responses (≥80% coverage). **Grocery**: Support multilingual names, perishable flags. **Debug**: Log slow queries (>300ms) to `django.db.backends`. **Deploy**: Docker image (`grocery/product-catalog`) to `kind` cluster via Helm chart (`charts/product-catalog`). |
| 2.2 Inventory | Build real-time stock management with perishable tracking and Kafka integration. | 2 weeks | 1.4, 1.5 | **Tech**: NestJS, Firestore emulator, Kafka (`bitnami/kafka:3.7`). **Model**: Firestore collection `inventory/{sku}/{location}` with fields `{stock: number, expires_at: timestamp}`. **API**: REST (`/stock/{sku}`, GET/POST, ≤100ms). **Kafka**: Deploy single-broker Kafka in Docker (`docker run -d bitnami/kafka`), create topic `order_events`. Consumer updates stock on `OrderPlaced`. **Logic**: Optimistic locking via Firestore transactions to prevent oversells. **Script**: Seed 10,000 SKUs (`scripts/seed_inventory.js`). **Tests**: Jest for API, transaction logic; mock Kafka with `kafkajs`. **Grocery**: Track expiration, flag low stock (<10 units). **Debug**: Log Firestore latency, Kafka consumer errors. **Deploy**: Docker image (`grocery/inventory`) to `kind`. **Resource**: Limit Kafka to 512MB RAM. |
| 2.3 Order | Build cart, checkout, and order tracking with substitution preferences. | 2 weeks | 2.1, 2.2 | **Tech**: Django, PostgreSQL. **Models**: `Order(id: uuid, user_id: str, items: jsonb, status: str, substitution_prefs: jsonb)`, `Cart(user_id: str, items: jsonb)`. **API**: REST (`/cart`, `/checkout`, `/orders/{id}`) with POST/GET (≤1s). **Kafka**: Publish `OrderPlaced` to `order_events`. **Logic**: Validate stock via Inventory API, store substitutions (e.g., `{sku: "milk", allow: true, alt_sku: "milk2"}`). **Script**: Mock checkout flow (`scripts/test_checkout.py`). **Tests**: pytest for cart, checkout, edge cases (e.g., out-of-stock). **Grocery**: Support per-item substitutions, order status (`pending`, `fulfilled`). **Debug**: Log checkout latency (>800ms). **Deploy**: Docker image (`grocery/order`) to `kind`. |
| 2.4 Payment | Integrate Stripe for secure payments. | 1 week | 2.3 | **Tech**: NestJS, Stripe SDK (`@stripe/stripe-js`). **Model**: PostgreSQL `Payment(id: uuid, order_id: uuid, stripe_id: str, status: str)`. **API**: REST (`/pay`, POST, ≤1s). **Kafka**: Publish `PaymentProcessed` to `payment_events`. **Logic**: Create Stripe PaymentIntent, handle webhooks (`payment_intent.succeeded`). **Script**: Test webhook handler (`scripts/test_stripe.js`). **Tests**: Jest for API, webhook parsing; mock Stripe with `stripe-mock`. **Grocery**: Ensure PCI DSS via Stripe.js. **Debug**: Log webhook failures. **Deploy**: Docker image (`grocery/payment`) to `kind`. **Resource**: Limit to 256MB RAM. |
| 2.5 User | Build JWT-based authentication and basic profiles. | 1 week | 1.4 | **Tech**: Django, `djangorestframework-simplejwt`. **Model**: PostgreSQL `User(id: uuid, email: str, password: str, role: str, name: str)`. **API**: REST (`/login`, `/profile`, POST/GET). **Logic**: Issue JWT tokens, RBAC with roles (`customer`, `admin`). **Script**: Create test users (`scripts/seed_users.py`). **Tests**: pytest for auth, role-based access. **Grocery**: Support multilingual profile names. **Debug**: Log auth failures (e.g., invalid credentials). **Deploy**: Docker image (`grocery/user`) to `kind`. **Resource**: Limit to 256MB RAM. |
| 2.6 Delivery | Integrate Grab/ViettelPost for same-day slots and pickup, using Redis cache. | 1.5 weeks | 2.3 | **Tech**: NestJS, Firestore, Redis (`redis:7.0-alpine`). **Model**: Firestore `delivery/{orderId}` with `{slot: timestamp, provider: str, status: str}`. **API**: REST (`/slots`, `/delivery/{orderId}`, GET/POST, ≤500ms). **Cache**: Store slots in Redis (`slots:{date}`) with 1-hour TTL. **Kafka**: Publish `DeliveryScheduled` to `delivery_events`. **Logic**: Mock Grab/ViettelPost APIs initially, fallback to next-day slots on failure. **Script**: Seed slots (`scripts/seed_slots.js`). **Tests**: Jest for slot booking, cache hits. **Grocery**: Prioritize same-day slots, support pickup. **Debug**: Log cache misses, API latency. **Deploy**: Docker image (`grocery/delivery`) to `kind`. |
| 2.7 Notification | Build email/SMS notifications for order and delivery updates. | 1 week | 2.3, 2.6 | **Tech**: NestJS, Firestore for logs, SendGrid/Twilio. **Model**: Firestore `notifications/{id}` with `{order_id: str, type: str, status: str}`. **API**: Internal (`/notify`, POST). **Kafka**: Consume `OrderConfirmed`, `DeliveryScheduled` from `order_events`, `delivery_events`. **Logic**: Send email/SMS via SendGrid/Twilio APIs. **Script**: Test notification flow (`scripts/test_notify.js`). **Tests**: Jest for message delivery, mock SendGrid/Twilio. **Grocery**: Include substitution details in notifications. **Debug**: Log delivery failures. **Deploy**: Docker image (`grocery/notification`) to `kind`. **Resource**: Limit to 256MB RAM. |

**Grocery Focus**:
- **Inventory**: Implement optimistic locking in Firestore to prevent oversells, track expiration dates for perishables, and flag low stock for restocking alerts.
- **Delivery**: Cache slots in Redis for ≤500ms responses, support same-day delivery with fallback logic, and include pickup options.
- **Order**: Include substitution preferences in checkout (e.g., `substitutionAllowed`, per-item alternates) and validate stock availability.
- **Automation**: Use scripts (`/scripts`) to seed data and test flows, reducing manual effort for solo development.
- **Testing**: Write unit tests for grocery-specific logic (e.g., oversell prevention, slot booking) to ensure ≥99% inventory accuracy and ≥98% slot success.

**Code Snippets**:
- **Product Catalog (Django Model)**:
  ```python
  # services/product-catalog/product/models.py
  from django.db import models

  class Product(models.Model):
      sku = models.CharField(max_length=50, unique=True)
      name_en = models.CharField(max_length=200)
      name_vi = models.CharField(max_length=200)
      category = models.CharField(max_length=100)
      price = models.DecimalField(max_digits=10, decimal_places=2)
      is_perishable = models.BooleanField(default=False)
      expires_at = models.DateTimeField(null=True, blank=True)

      class Meta:
          indexes = [models.Index(fields=['name_en', 'name_vi'], name='name_idx')]
  ```
- **Inventory (NestJS Service)**:
  ```typescript
  // services/inventory/src/stock.service.ts
  import { Injectable } from '@nestjs/common';
  import * as admin from 'firebase-admin';

  @Injectable()
  export class StockService {
    private db = admin.firestore();

    async getStock(sku: string): Promise<any> {
      const doc = await this.db.collection('inventory').doc(sku).get();
      if (!doc.exists) throw new Error('SKU not found');
      return doc.data();
    }

    async updateStock(sku: string, quantity: number): Promise<void> {
      await this.db.runTransaction(async (t) => {
        const ref = this.db.collection('inventory').doc(sku);
        const doc = await t.get(ref);
        const stock = doc.data()?.stock || 0;
        if (stock < quantity) throw new Error('Insufficient stock');
        t.update(ref, { stock: stock - quantity });
      });
    }
  }
  ```
- **Order (Checkout API)**:
  ```python
  # services/order/order/views.py
  from rest_framework.views import APIView
  from rest_framework.response import Response
  import requests

  class CheckoutView(APIView):
      def post(self, request):
          user_id = request.user.id
          items = request.data['items']  # [{sku, quantity, substitution_prefs}]
          for item in items:
              stock = requests.get(f'http://inventory:3000/stock/{item["sku"]}').json()
              if stock['stock'] < item['quantity']:
                  raise ValueError('Out of stock')
          order = Order.objects.create(user_id=user_id, items=items, substitution_prefs=request.data.get('substitution_prefs'))
          # Publish to Kafka
          return Response({'order_id': order.id}, status=201)
  ```

**Resource Considerations**:
- **Laptop Specs**: Use ~6–8GB RAM for Docker containers (512MB each for services, 2GB for dev container, 2GB for `kind`, 512MB for Kafka).
- **Optimization**: Run only active service containers during development (e.g., `docker stop inventory` when testing Order). Use `docker stats` to monitor.
- **Kafka**: Deploy single-broker Kafka in Task 2.2 with 512MB RAM to minimize overhead. Scale to 3 brokers in Phase 6.
- **Testing**: Run tests locally in dev container to avoid CI/CD overhead. Use `pytest --cov` and `jest --coverage` for quick feedback.

**Automation Scripts**:
- **Seed Products**:
  ```python
  # scripts/seed_products.py
  from product.models import Product
  from faker import Faker
  fake = Faker()

  for i in range(10000):
      Product.objects.create(
          sku=f"SKU{i}",
          name_en=fake.word(),
          name_vi=fake.word(),
          category=fake.word(),
          price=fake.random_int(1, 100),
          is_perishable=fake.random_element([True, False]),
          expires_at=fake.date_time() if is_perishable else None
      )
  ```
- **Test Slots**:
  ```javascript
  // scripts/seed_slots.js
  const redis = require('redis');
  const client = redis.createClient({ url: 'redis://localhost:6379' });
  await client.connect();
  await client.setEx('slots:2025-05-15', 3600, JSON.stringify([
    { time: '10:00', available: true },
    { time: '12:00', available: false }
  ]));
  await client.quit();
  ```

**Output**: Functional MVP microservices with REST APIs, Kafka integration, and grocery-specific features (e.g., perishable tracking, substitutions, same-day slots). Docker images deployed to `kind` cluster, ready for frontend integration in Phase 3.

### Phase 3: Frontend and UX (3–4 Weeks)
**Goal**: Develop a mobile-first frontend with grocery-specific UX (search, checkout, delivery slots).

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 3.1 Prototype | Design UX in Figma for search, cart, checkout, slots. | 1 week | None | Mobile-first, multilingual (English/Vietnamese). Single-page checkout, calendar slot picker. |
| 3.2 Build frontend | Develop React frontend, connect to GraphQL/REST APIs. | 2 weeks | 2.1–2.7, 3.1 | Use Apollo Client for GraphQL. Components: `ProductList`, `Cart`, `SlotPicker`. Cache with Apollo. |
| 3.3 Multilingual | Implement English/Vietnamese toggle. | 3 days | 3.2 | Use `react-i18next`. Fetch translations from Product Catalog API. |
| 3.4 Optimize | Ensure ≤2s page loads with Lighthouse. | 3 days | 3.2 | Compress images, use Cloudflare CDN, lazy-load products. |

**Grocery Focus**: Streamlined checkout with substitution options, clear perishable indicators, and fast slot selection (≤500ms).
**Code Snippet (GraphQL Query)**:
```graphql
query GetProducts($category: String!) {
  products(category: $category) {
    sku
    nameEn
    nameVi
    price
    isPerishable
  }
}
```
**Output**: Responsive frontend integrated with backend, supporting MVP flows.

### Phase 4: Third-Party Integrations (2–3 Weeks)
**Goal**: Finalize Stripe and Grab/ViettelPost integrations, prepare for future providers (e.g., MoMo).

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 4.1 Stripe | Complete Stripe.js integration, test webhooks. | 1 week | 2.4 | NestJS, `@stripe/stripe-js`. Kafka topic: `payment_events`. Test with Stripe sandbox. |
| 4.2 Grab/ViettelPost | Integrate delivery APIs, test slot booking. | 1.5 weeks | 2.6 | NestJS, REST APIs. Cache slots in Redis. Test with provider sandboxes. |
| 4.3 Extensibility | Abstract Payment/Delivery Services for MoMo, Gojek. | 3 days | 4.1, 4.2 | Strategy pattern in NestJS: `PaymentProcessor`, `DeliveryProvider`. Store configs in Vault. |

**Grocery Focus**: Ensure ≤1s payment processing, ≤500ms slot updates, and reliable webhook handling for delivery tracking.
**Output**: Fully integrated third-party services, extensible for future providers.

### Phase 5: Testing and Validation (3–4 Weeks)
**Goal**: Ensure reliability, performance, and grocery-specific functionality through comprehensive testing.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 5.1 Unit tests | Write tests for all services (≥80% coverage). | 1 week | 2.1–2.7 | pytest for Django, Jest for NestJS. Example: Test Inventory stock locking. |
| 5.2 Integration tests | Test service interactions (REST, Kafka). | 1 week | 2.1–2.7, 4.1–4.3 | Testcontainers for PostgreSQL/Firestore. Mock Stripe/Grab APIs with `nock`. |
| 5.3 E2E tests | Test user flows (search, checkout, slots). | 1 week | 3.2, 5.2 | Cypress for browser tests. Test multilingual UX, substitutions. |
| 5.4 Load tests | Simulate 300,000 users, validate performance. | 3 days | 5.2 | Locust: Test ≤500ms search, ≤1s checkout. Monitor with Prometheus/Grafana. |
| 5.5 Security tests | Scan for vulnerabilities, test JWT/RBAC. | 2 days | 5.2 | OWASP ZAP for APIs, Snyk for dependencies. |

**Grocery Focus**: Test inventory accuracy (≥99%), slot booking success (≥98%), and substitution flows under load.
**Output**: Fully tested platform meeting performance and reliability goals.

### Phase 6: Deployment and Beta Launch (2–3 Weeks)
**Goal**: Deploy the MVP, conduct beta testing, and prepare for production.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 6.1 Staging deploy | Deploy to staging Kubernetes cluster. | 3 days | 2.1–2.7, 3.2, 4.1–4.3 | Use Helm: `helm upgrade grocery ./charts`. Blue-green for Inventory/Delivery. |
| 6.2 Beta testing | Launch to 100–500 users, collect feedback. | 1 week | 6.1, 5.1–5.5 | Use Django `waffle` for feature flags. Google Forms, Hotjar for feedback. |
| 6.3 Fix issues | Address beta feedback (e.g., UX, performance). | 1 week | 6.2 | Prioritize critical issues (e.g., slow checkout >1s). Update GitHub Issues. |
| 6.4 Production deploy | Deploy to production, monitor KPIs. | 3 days | 6.3 | Blue-green deployment, smoke tests (Postman). Monitor with Prometheus/Grafana. |

**Grocery Focus**: Validate real-time inventory (≤100ms), delivery slots (≤500ms), and substitution UX during beta.
**Output**: Production-ready MVP, initial KPIs (e.g., conversion rate ≥3%).

### Phase 7: Post-MVP Enhancements (6–8 Weeks, Optional)
**Goal**: Add deferred features and scale for growth.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 7.1 Promotions | Build discount/coupon service. | 2 weeks | 2.3 | Django, PostgreSQL: `Promotion(id, type, discount)`. API: `/promotions`. |
| 7.2 Search | Implement advanced search with Elasticsearch. | 2 weeks | 2.1 | NestJS, Elasticsearch: Index 50,000 SKUs, support perishable filters. |
| 7.3 Recommendation | Build personalized suggestions. | 2 weeks | 2.1, 2.3 | Django, PostgreSQL: `Recommendation(user_id, skus)`. Basic collaborative filtering. |
| 7.4 MoMo/Gojek | Integrate MoMo payments, Gojek delivery. | 2 weeks | 4.3 | NestJS, REST APIs. Update Payment/Delivery abstractions. |

**Grocery Focus**: Promotions for near-expiry perishables, search filters for dietary needs, and expanded delivery capacity.
**Output**: Enhanced platform with advanced features, ready for broader rollout.

## Timeline
- **Total Duration**: 25–34 weeks (6–8 months) for MVP (Phases 1–6), plus 6–8 weeks for post-MVP (Phase 7).
- **Breakdown**:
  - Phase 1: 1–2 weeks
  - Phase 2: 8–10 weeks
  - Phase 3: 3–4 weeks
  - Phase 4: 2–3 weeks
  - Phase 5: 3–4 weeks
  - Phase 6: 2–3 weeks
  - Phase 7: 6–8 weeks (optional)
- **Assumption**: 20–30 hours/week, with flexibility for adjustments (e.g., faster progress with automation).

## Best Practices for Solo Development
- **Prioritization**: Focus on MVP (Product Catalog, Inventory, Order, Payment, User, Delivery, Notification) to launch quickly. Defer Promotions, Search, Recommendation.
- **Automation**:
  - Use GitHub Actions for CI/CD (lint, test, deploy).
  - Deploy monitoring (Prometheus/Grafana, Loki) early to catch issues.
  - Script repetitive tasks (e.g., Firestore seeding) in `/scripts`.
- **Task Management**: Use GitHub Projects with Kanban board (To-Do, In Progress, Done). Label tasks (e.g., `priority:high`, `grocery`) and set milestones (e.g., “MVP”).
- **Technical Debt**:
  - Allocate 10% of time to refactoring (e.g., optimize Inventory queries).
  - Use Snyk to catch vulnerabilities early.
- **Grocery-Specific**:
  - Test inventory locking and substitution flows rigorously to prevent oversells.
  - Ensure delivery slot APIs are cached (Redis) for ≤500ms responses.
  - Validate multilingual UX (English/Vietnamese) in beta.
- **Cloud Prep**: Use cloud-agnostic tools (e.g., MinIO, Helm) and test staging deployments for AWS/GCP compatibility.

## Monitoring and Validation
- **KPIs**: Track during beta and production (Phase 6):
  - Conversion Rate: ≥3%.
  - Order Fulfillment: ≥95%.
  - Inventory Accuracy: ≥99%.
  - Slot Booking Success: ≥98%.
  - Uptime: ≥99.9%.
- **Tools**: Prometheus/Grafana for metrics, GA4 for user behavior, Hotjar for UX, Firestore for grocery events.
- **Feedback**: Collect beta feedback via Google Forms/Hotjar to refine UX (e.g., checkout, slots).

## Risks and Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| Overrunning timeline | Delays launch | Prioritize MVP, automate tests, use GitHub Projects for tracking. |
| Performance issues | Slow UX (e.g., >1s checkout) | Cache with Redis, test with Locust, monitor with Prometheus. |
| Third-party failures | Payment/delivery disruptions | Use circuit breakers, mock APIs in tests, cache responses. |
| Technical debt | Maintenance burden | Refactor incrementally, enforce coverage ≥80%, use Snyk. |

## Next Steps
- Start Phase 1 (setup) by initializing the repo and Kubernetes cluster.
- Review tasks in GitHub Projects weekly, adjusting priorities as needed.
- Deploy monitoring early (Phase 1) to track progress and catch issues.
- Revisit plan post-MVP to prioritize post-MVP features (e.g., Promotions, MoMo).
