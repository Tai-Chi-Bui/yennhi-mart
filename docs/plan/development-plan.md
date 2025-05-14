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
| 1.6 Configure CI/CD | Create GitHub Actions workflow for build/test, using Docker Hub. | 2 days | 1.1 | Workflow: lint (Flake8, ESLint), test (pytest, Jest 네이버재팬), build Docker images. Example: `docker build -t grocery/inventory:test`. Defer Harbor to Phase 6. See `.github/workflows/ci.yml`. |
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
**Goal**: Build a fully fledged set of MVP microservices (Product Catalog, Inventory, Order, Payment, User, Delivery, Notification) for a large-scale grocery e-commerce platform, ensuring scalability, performance, and grocery-specific functionality. Each service includes comprehensive APIs, database models, Kafka event handling, caching, rate limiting, circuit breakers, health checks, and automated tests, leveraging Phase 1’s Docker environment (PostgreSQL, Firestore emulator, Redis, `kind` cluster). The implementation is optimized for a solo developer on a work laptop (~8–12GB RAM, 4 cores), with detailed automation, logging, and metrics to support 100,000 DAU and 300,000 peak users.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 2.1 Product Catalog | Build a scalable service to manage 50,000 SKUs with multilingual descriptions, categories, and basic search. | 1.5 weeks | 1.4 | **Tech**: Django 4.2, PostgreSQL (`postgres:14-alpine`), `django-environ`. **Models**: `Product(sku: str, name_en: str, name_vi: str, category_id: fk, price: decimal, is_perishable: bool, expires_at: datetime, stock_updated_at: datetime)`, `Category(id: uuid, name_en: str, name_vi: str)`. **API**: REST (`/products`, `/products/{sku}`, `/categories`, GET) with pagination (`limit`, `offset`), filtering (`category`, `is_perishable`). **Search**: PostgreSQL GIN index on `name_en`, `name_vi` for ≤500ms queries; use `tsvector` for full-text search. **Caching**: Redis (`products:{sku}`, 1-hour TTL) for hot SKUs. **Rate Limiting**: NGINX (`limit_req zone=products burst=50`). **Health Check**: `/health` endpoint (checks DB, Redis). **Kafka**: Publish `ProductUpdated` (e.g., price change) to `product_events`. **Scripts**: Seed 50,000 SKUs (`scripts/seed_products.py`, 20% perishables), generate categories (`scripts/seed_categories.py`). **Tests**: pytest (`pytest-django`) for models, APIs, search (≥85% coverage); Testcontainers for PostgreSQL. **Metrics**: Prometheus (`django_prometheus`) for API latency, cache hits. **Logging**: Log slow queries (>300ms) to `django.db.backends`, API errors to `sentry-sdk`. **Grocery**: Support multilingual names, perishable flags, expiration alerts (e.g., expires < 3 days). **Scalability**: Shard PostgreSQL by `category_id` (pre-sharded for 50,000 SKUs). **HA**: Deploy 2 replicas in `kind` via Helm (`charts/product-catalog`, `replicas: 2`). **Debug**: Use Django Debug Toolbar locally. **CI/CD**: GitHub Actions (`lint`, `test`, `build`, `push` to Docker Hub). **Docker**: Image (`grocery/product-catalog:1.0`, ~200MB). **Resource**: 512MB RAM per container. |
| 2.2 Inventory | Build a real-time stock management service with perishable tracking, optimistic locking, and Kafka integration. | 2 weeks | 1.4, 1.5 | **Tech**: NestJS 10, Firestore emulator, Kafka (`bitnami/kafka:3.7`), `nestjs-throttler`. **Model**: Firestore `inventory/{sku}/{location}` with `{stock: number, expires_at: timestamp, last_updated: timestamp, low_stock_threshold: number}`. **API**: REST (`/stock/{sku}`, GET/POST, ≤100ms; `/stock/batch`, POST for bulk updates). **Kafka**: Deploy single-broker Kafka in Docker (`docker run -d --memory=512m bitnami/kafka`), topics: `order_events`, `product_events`. Consume `OrderPlaced` to deduct stock, `ProductUpdated` to sync SKUs. **Logic**: Optimistic locking via Firestore transactions; alert on low stock (<`low_stock_threshold`, default 10). **Caching**: Redis (`stock:{sku}`, 30-min TTL) for GET requests. **Rate Limiting**: `nestjs-throttler` (100 req/s). **Circuit Breaker**: `opossum` for Firestore calls (timeout 200ms). **Health Check**: `/health` (Firestore, Kafka, Redis). **Scripts**: Seed 50,000 SKUs (`scripts/seed_inventory.js`), simulate stock updates (`scripts/test_stock.js`). **Tests**: Jest for APIs, transactions, Kafka consumers (≥85% coverage); mock Kafka with `kafkajs`, Firestore with emulator. **Metrics**: Prometheus (`prom-client`) for stock update latency, error rates. **Logging**: Winston to Loki (deferred to Phase 5), console for local debugging. **Grocery**: Track expiration (auto-remove expired stock), support multi-location stock (e.g., warehouse, store). **Scalability**: Partition Firestore by `location` for 300,000 peak users. **HA**: 2 replicas in `kind` (`charts/inventory`). **Debug**: Trace Firestore transactions with OpenTelemetry (deferred to Phase 5). **CI/CD**: GitHub Actions (`test`, `build`, `push`). **Docker**: Image (`grocery/inventory:1.0`, ~150MB). **Resource**: 512MB RAM (Kafka 512MB). |
| 2.3 Order | Build a cart, checkout, and order tracking service with substitution preferences and real-time validation. | 2 weeks | 2.1, 2.2 | **Tech**: Django, PostgreSQL, `django-redis`, `requests`. **Models**: `Order(id: uuid, user_id: str, items: jsonb, status: str, substitution_prefs: jsonb, created_at: datetime)`, `Cart(user_id: str, items: jsonb, updated_at: datetime)`. **API**: REST (`/cart`, `/cart/{user_id}`, `/checkout`, `/orders/{id}`, GET/POST, ≤1s). **Kafka**: Publish `OrderPlaced` to `order_events` (payload: `{order_id, items, substitution_prefs}`). **Logic**: Validate stock via Inventory API (`/stock/{sku}`), apply substitutions (e.g., `{sku: "milk", allow: true, alt_sku: "milk2"}`), update `status` (`pending`, `confirmed`, `fulfilled`). **Caching**: Redis (`cart:{user_id}`, 24-hour TTL). **Rate Limiting**: NGINX (`limit_req zone=order burst=20`). **Circuit Breaker**: `requests` with `tenacity` (retry 3x, timeout 500ms) for Inventory API. **Health Check**: `/health` (DB, Redis, Inventory). **Scripts**: Mock checkout (`scripts/test_checkout.py`), simulate substitutions (`scripts/test_substitutions.py`). **Tests**: pytest for cart, checkout, substitutions (≥85% coverage); Testcontainers for PostgreSQL. **Metrics**: Prometheus for checkout latency, order volume. **Logging**: Log checkout failures, substitution errors to Sentry. **Grocery**: Support per-item substitutions, validate stock in real-time, notify on out-of-stock. **Scalability**: Index `user_id` for fast cart retrieval. **HA**: 2 replicas in `kind` (`charts/order`). **Debug**: Log API call latencies (>800ms). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/order:1.0`, ~200MB). **Resource**: 512MB RAM. |
| 2.4 Payment | Integrate Stripe for secure, scalable payments with webhook handling. | 1 week | 2.3 | **Tech**: NestJS, Stripe SDK (`@stripe/stripe-js`), PostgreSQL. **Model**: `Payment(id: uuid, order_id: uuid, stripe_id: str, amount: decimal, status: str, created_at: datetime)`. **API**: REST (`/pay`, POST, ≤1s; `/payment/{order_id}`, GET). **Kafka**: Publish `PaymentProcessed` to `payment_events` (payload: `{order_id, status}`). **Logic**: Create Stripe PaymentIntent, handle webhooks (`payment_intent.succeeded`, `payment_intent.failed`) with idempotency. **Security**: PCI DSS via Stripe.js, store no card data. **Rate Limiting**: `nestjs-throttler` (50 req/s). **Circuit Breaker**: `opossum` for Stripe API (timeout 500ms). **Health Check**: `/health` (DB, Stripe). **Scripts**: Test webhook handler (`scripts/test_stripe.js`), simulate payments (`scripts/simulate_payment.js`). **Tests**: Jest for APIs, webhooks (≥85% coverage); mock Stripe with `stripe-mock`. **Metrics**: Prometheus for payment success rate, latency. **Logging**: Log webhook errors to Sentry. **Grocery**: Ensure fast payment processing for quick checkout. **Scalability**: Handle 300,000 peak users with async webhooks. **HA**: 2 replicas in `kind` (`charts/payment`). **Debug**: Log Stripe API errors. **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/payment:1.0`, ~150MB). **Resource**: 256MB RAM. |
| 2.5 User | Build JWT-based authentication, user profiles, and RBAC for secure access. | 1 week | 1.4 | **Tech**: Django, `djangorestframework-simplejwt`, PostgreSQL. **Model**: `User(id: uuid, email: str, password: str, role: str, name_en: str, name_vi: str, created_at: datetime)`. **API**: REST (`/login`, `/refresh`, `/profile`, `/users/{id}`, POST/GET). **Logic**: Issue JWT tokens (access: 15m, refresh: 24h), RBAC with roles (`customer`, `admin`, `staff`). **Security**: Hash passwords with `argon2`, enforce TLS 1.3. **Rate Limiting**: NGINX (`limit_req zone=user burst=100`). **Health Check**: `/health` (DB). **Scripts**: Seed test users (`scripts/seed_users.py`), test RBAC (`scripts/test_rbac.py`). **Tests**: pytest for auth, RBAC, profile updates (≥85% coverage). **Metrics**: Prometheus for login success rate, token issuance. **Logging**: Log auth failures to Sentry. **Grocery**: Support multilingual names, role-based order access (e.g., staff view pending orders). **Scalability**: Index `email` for fast login. **HA**: 2 replicas in `kind` (`charts/user`). **Debug**: Log token validation errors. **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/user:1.0`, ~200MB). **Resource**: 256MB RAM. |
| 2.6 Delivery | Integrate Grab/ViettelPost for same-day delivery slots, in-store pickup, and real-time tracking. | 1.5 weeks | 2.3 | **Tech**: NestJS, Firestore, Redis (`redis:7.0-alpine`), `nestjs-throttler`. **Model**: Firestore `delivery/{orderId}` with `{slot: timestamp, provider: str, status: str, tracking_url: str, pickup: bool}`. **API**: REST (`/slots`, `/delivery/{orderId}`, `/pickup`, GET/POST, ≤500ms). **Caching**: Redis (`slots:{date}`, 1-hour TTL) for slot availability. **Kafka**: Publish `DeliveryScheduled`, `DeliveryUpdated` to `delivery_events`. **Logic**: Mock Grab/ViettelPost APIs (use sandbox in Phase 4), fallback to next-day slots on failure, support pickup locations. **Rate Limiting**: `nestjs-throttler` (200 req/s). **Circuit Breaker**: `opossum` for provider APIs (timeout 300ms). **Health Check**: `/health` (Firestore, Redis). **Scripts**: Seed slots (`scripts/seed_slots.js`), test pickup (`scripts/test_pickup.js`). **Tests**: Jest for slot booking, cache hits (≥85% coverage). **Metrics**: Prometheus for slot booking success, latency. **Logging**: Log cache misses, API errors to Sentry. **Grocery**: Prioritize same-day slots, display tracking URLs. **Scalability**: Partition Firestore by `orderId`. **HA**: 2 replicas in `kind` (`charts/delivery`). **Debug**: Log provider API latency (>400ms). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/delivery:1.0`, ~150MB). **Resource**: 512MB RAM. |
| 2.7 Notification | Build scalable email/SMS notifications for order and delivery updates. | 1 week | 2.3, 2.6 | **Tech**: NestJS, Firestore, SendGrid/Twilio, `nestjs-throttler`. **Model**: Firestore `notifications/{id}` with `{order_id: str, type: str, status: str, sent_at: timestamp, payload: object}`. **API**: Internal (`/notify`, POST). **Kafka**: Consume `OrderConfirmed`, `DeliveryScheduled`, `PaymentProcessed` from `order_events`, `delivery_events`, `payment_events`. **Logic**: Send multilingual email/SMS with substitution details (e.g., “Milk replaced with Milk2”). **Rate Limiting**: `nestjs-throttler` (50 req/s). **Circuit Breaker**: `opossum` for SendGrid/Twilio APIs (timeout 500ms). **Health Check**: `/health` (Firestore). **Scripts**: Test notifications (`scripts/test_notify.js`), simulate failures (`scripts/test_notify_fail.js`). **Tests**: Jest for message delivery, payload parsing (≥85% coverage); mock SendGrid/Twilio. **Metrics**: Prometheus for notification success rate. **Logging**: Log delivery failures to Sentry. **Grocery**: Include order status, substitution details in messages. **Scalability**: Batch notifications in Kafka consumer. **HA**: 2 replicas in `kind` (`charts/notification`). **Debug**: Log retry attempts. **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/notification:1.0`, ~150MB). **Resource**: 256MB RAM. |

**Grocery Focus**:
- **Inventory**: Prevent oversells with Firestore transactions, track perishable expiration (auto-remove expired stock), and alert on low stock for restocking.
- **Order**: Support complex substitution workflows (e.g., per-item alternates, user-defined priorities), validate stock in real-time, and handle partial fulfillment.
- **Delivery**: Optimize same-day slot booking with Redis caching, provide tracking URLs, and support in-store pickup with location-based filtering.
- **Notification**: Deliver multilingual, context-aware messages (e.g., “Order confirmed, milk substituted”) to enhance user trust.
- **Performance**: Ensure ≤100ms inventory updates, ≤500ms slot responses, ≤1s checkout with caching and circuit breakers.
- **Reliability**: Achieve ≥99% inventory accuracy and ≥98% slot booking success with robust error handling and retries.

**Code Snippets**:
- **Product Catalog (Django API)**:
  ```python
  # services/product-catalog/product/views.py
  from rest_framework import viewsets
  from .models import Product
  from .serializers import ProductSerializer
  from django.contrib.postgres.search import SearchVector

  class ProductViewSet(viewsets.ModelViewSet):
      queryset = Product.objects.all()
      serializer_class = ProductSerializer

      def get_queryset(self):
          queryset = super().get_queryset()
          query = self.request.query_params.get('search')
          if query:
              queryset = queryset.annotate(
                  search=SearchVector('name_en', 'name_vi')
              ).filter(search=query)
          return queryset[:100]  # Paginate
  ```
- **Inventory (NestJS API with Kafka)**:
  ```typescript
  // services/inventory/src/stock.controller.ts
  import { Controller, Get, Post, Param, Body } from '@nestjs/common';
  import { StockService } from './stock.service';
  import { Kafka } from 'kafkajs';

  @Controller('stock')
  export class StockController {
    private kafka = new Kafka({ brokers: ['kafka:9092'] });
    private producer = this.kafka.producer();

    constructor(private readonly stockService: StockService) {
      this.producer.connect();
    }

    @Get(':sku')
    async getStock(@Param('sku') sku: string) {
      return this.stockService.getStock(sku); // ≤100ms
    }

    @Post('update')
    async updateStock(@Body() { sku, quantity }: { sku: string; quantity: number }) {
      await this.stockService.updateStock(sku, quantity);
      await this.producer.send({
        topic: 'order_events',
        messages: [{ value: JSON.stringify({ event: 'StockUpdated', sku, quantity }) }],
      });
    }
  }
  ```
- **Order (Substitution Logic)**:
  ```python
  # services/order/order/services.py
  import requests
  from .models import Order
  from django.db import transaction

  def process_checkout(user_id, items, substitution_prefs):
      with transaction.atomic():
          for item in items:
              stock = requests.get(f'http://inventory:3000/stock/{item["sku"]}').json()
              if stock['stock'] < item['quantity']:
                  if substitution_prefs.get(item['sku'], {}).get('allow'):
                      item['sku'] = substitution_prefs[item['sku']]['alt_sku']
                      stock = requests.get(f'http://inventory:3000/stock/{item["sku"]}').json()
                      if stock['stock'] < item['quantity']:
                          raise ValueError('Substitution out of stock')
                  else:
                      raise ValueError('Out of stock')
          order = Order.objects.create(user_id=user_id, items=items, substitution_prefs=substitution_prefs)
          requests.post('http://inventory:3000/stock/update', json=items)
          return order
  ```
- **Delivery (Slot Caching)**:
  ```typescript
  // services/delivery/src/slot.service.ts
  import { Injectable } from '@nestjs/common';
  import { Redis } from 'ioredis';

  @Injectable()
  export class SlotService {
    private redis = new Redis('redis://redis:6379');

    async getSlots(date: string): Promise<any[]> {
      const cached = await this.redis.get(`slots:${date}`);
      if (cached) return JSON.parse(cached);
      const slots = await this.fetchSlotsFromProvider(date); // Mock or Grab/ViettelPost
      await this.redis.setex(`slots:${date}`, 3600, JSON.stringify(slots));
      return slots;
    }
  }
  ```

**Automation Scripts**:
- **Seed Products**:
  ```python
  # scripts/seed_products.py
  from product.models import Product, Category
  from faker import Faker
  from django.utils import timezone
  fake = Faker()

  categories = [Category.objects.create(name_en=fake.word(), name_vi=fake.word()) for _ in range(50)]
  for i in range(50000):
      is_perishable = fake.random_element([True, False])
      Product.objects.create(
          sku=f"SKU{i:06d}",
          name_en=fake.word(),
          name_vi=fake.word(),
          category=fake.random_element(categories),
          price=fake.random_int(1, 1000) / 100,
          is_perishable=is_perishable,
          expires_at=fake.date_time_between(start_date='now', end_date='+30d') if is_perishable else None
      )
  ```
- **Test Checkout**:
  ```python
  # scripts/test_checkout.py
  import requests
  payload = {
      'user_id': 'user123',
      'items': [{'sku': 'SKU000001', 'quantity': 2}],
      'substitution_prefs': {'SKU000001': {'allow': True, 'alt_sku': 'SKU000002'}}
  }
  response = requests.post('http://localhost:8000/checkout', json=payload)
  print(response.json())
  ```

**Resource Considerations**:
- **Laptop Specs**: Allocate ~8GB RAM for Docker containers (512MB each for Product Catalog, Inventory, Order, Delivery; 256MB for Payment, User, Notification; 512MB for Kafka; 2GB for dev container; 2GB for `kind`).
- **Optimization**: Run only active service containers (e.g., `docker stop payment` when testing Inventory). Use `docker stats` to monitor CPU/RAM.
- **Kafka**: Single-broker Kafka (512MB) for local development; scale to 3 brokers in Phase 6.
- **Testing**: Run tests in dev container to minimize CI/CD overhead. Use `pytest --cov=product` and `jest --coverage` for quick feedback.
- **Constraints**: Ensure laptop’s Docker Desktop or Engine supports ~10 containers. Use WSL2 (Windows) for better performance if needed.

**Output**: Fully functional MVP microservices with scalable REST APIs, Kafka-driven event handling, Redis caching, and grocery-specific features (e.g., perishable tracking, substitutions, same-day slots). Docker images deployed to `kind` cluster via Helm, integrated with CI/CD, and tested for ≥85% coverage, ready for frontend development in Phase 3.

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
