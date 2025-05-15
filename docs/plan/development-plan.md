# Development Plan

## Overview
This plan outlines the development of a large-scale, microservices-based grocery e-commerce platform supporting 100,000 daily active users (DAU), 300,000 peak users, and 50,000 SKUs. The platform prioritizes scalability, performance (≤500ms search, ≤2s page load, ≤100ms inventory updates, 99.9% uptime), and grocery-specific features (e.g., real-time inventory, same-day delivery, substitutions, multilingual English/Vietnamese UX). It runs on Kubernetes/Docker on-premises, with future cloud migration (e.g., AWS EKS). As a solo developer, the plan focuses on automation, prioritization, and modularity to streamline implementation.

## Scope
- **MVP Features**:
  - Product Catalog: Browse/search 50,000 SKUs, multilingual descriptions, dietary/expiry filters, popular items.
  - Inventory: Real-time stock updates (≤100ms), perishable tracking, substitutions.
  - Order: Cart, checkout (≤1s), order tracking, SKU-based discounts.
  - Payment: Stripe integration, PCI DSS compliant, extensible for future providers.
  - User: JWT-based authentication, basic profiles.
  - Delivery: Grab/ViettelPost integration, same-day slots (≤500ms), in-store pickup, extensible for future providers.
  - Notifications: Basic order/delivery updates (email/SMS).
- **Post-MVP Features** (Deferred):
  - Promotions: Advanced discounts, coupons, perishable-specific deals.
  - Search: Elasticsearch-based advanced filtering.
  - Recommendation: Personalized suggestions with collaborative filtering.
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
- **Third-Party**: Stripe (payments), Grab/ViettelPost (delivery), SendGrid/Twilio (notifications).

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

### Phase 2: Core Microservices Development (9–11 Weeks)
**Goal**: Build a fully fledged set of MVP microservices (Product Catalog, Inventory, Order, Payment, User, Delivery, Notification) with enhanced features (SKU-based discounts, dietary/expiry filters, popular items, extensible Payment/Delivery) for a large-scale grocery e-commerce platform. Ensure scalability, performance, and grocery-specific functionality (e.g., perishable tracking, substitutions, same-day delivery). Each service includes comprehensive APIs, database models, Kafka event handling, caching, rate limiting, circuit breakers, health checks, and automated tests, leveraging Phase 1’s Docker environment. The implementation is optimized for a solo developer, with automation and modularity to support 100,000 DAU and 300,000 peak users.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 2.1 Product Catalog | Build a scalable service for 50,000 SKUs with multilingual descriptions, basic search, enhanced filters (dietary, expires_soon), and popular items. | 2 weeks | 1.4 | **Tech**: Django 4.2, PostgreSQL, `django-environ`, `django-filter`. **Models**: `Product(sku: str, name_en: str, name_vi: str, category_id: fk, price: decimal, is_perishable: bool, expires_at: datetime, stock_updated_at: datetime, dietary: array[str], tags: array[str])`, `Category(id: uuid, name_en: str, name_vi: str)`. **API**: REST (`/products`, `/products/{sku}`, `/categories`, `/products/popular`, GET) with pagination (`limit`, `offset`), filtering (`category`, `is_perishable`, `dietary`, `expires_soon`, `tags`). **Search**: PostgreSQL GIN index on `name_en`, `name_vi`, `dietary`, `tags` for ≤500ms queries; use `tsvector` for full-text search; `expires_soon` filter (`expires_at < now() + 3 days`). **Popular Items**: Aggregate via `SELECT sku, COUNT(*) FROM order_items WHERE created_at > now() - interval '7 days' GROUP BY sku ORDER BY COUNT DESC LIMIT 10`. **Caching**: Redis (`products:{sku}`, `popular:{category}`, 1-hour TTL). **Rate Limiting**: NGINX (`limit_req zone=products burst=50`). **Health Check**: `/health` (DB, Redis). **Kafka**: Publish `ProductUpdated` (price/dietary changes) to `product_events`. **Scripts**: Seed 50,000 SKUs (`seed_products.py`, 20% perishables, 10% vegan/gluten-free), categories (`seed_categories.py`). **Tests**: pytest (`pytest-django`) for models, APIs, filters, popular items (≥85% coverage); Testcontainers for PostgreSQL. **Metrics**: Prometheus (`django_prometheus`) for API latency, cache hits, filter usage. **Logging**: Log slow queries (>300ms) to `django.db.backends`, API errors to `sentry-sdk`. **Grocery**: Support multilingual names, perishable flags, dietary filters, near-expiry highlights, popular items on homepage. **Scalability**: Shard PostgreSQL by `category_id`; index `dietary`, `tags`. **HA**: Deploy 2 replicas in `kind` via Helm (`charts/product-catalog`). **Debug**: Use Django Debug Toolbar locally. **CI/CD**: GitHub Actions (`lint`, `test`, `build`, `push`). **Docker**: Image (`grocery/product-catalog:1.0`, ~200MB). **Resource**: 512MB RAM. |
| 2.2 Inventory | Build a real-time stock management service with perishable tracking, optimistic locking, and Kafka integration. | 2 weeks | 1.4, 1.5 | **Tech**: NestJS 10, Firestore emulator, Kafka (`bitnami/kafka:3.7`), `nestjs-throttler`. **Model**: Firestore `inventory/{sku}/{location}` with `{stock: number, expires_at: timestamp, last_updated: timestamp, low_stock_threshold: number}`. **API**: REST (`/stock/{sku}`, GET/POST, ≤100ms; `/stock/batch`, POST). **Kafka**: Deploy single-broker Kafka in Docker (`docker run -d --memory=512m bitnami/kafka`), topics: `order_events`, `product_events`. Consume `OrderPlaced` to deduct stock, `ProductUpdated` to sync SKUs. **Logic**: Optimistic locking via Firestore transactions; alert on low stock (<`low_stock_threshold`). **Caching**: Redis (`stock:{sku}`, 30-min TTL). **Rate Limiting**: `nestjs-throttler` (100 req/s). **Circuit Breaker**: `opossum` for Firestore calls (timeout 200ms). **Health Check**: `/health` (Firestore, Kafka, Redis). **Scripts**: Seed 50,000 SKUs (`seed_inventory.js`), simulate stock updates (`test_stock.js`). **Tests**: Jest for APIs, transactions, Kafka consumers (≥85% coverage); mock Kafka with `kafkajs`, Firestore with emulator. **Metrics**: Prometheus (`prom-client`) for stock update latency, error rates. **Logging**: Winston to Loki (deferred to Phase 5), console locally. **Grocery**: Track expiration, support multi-location stock. **Scalability**: Partition Firestore by `location`. **HA**: 2 replicas in `kind` (`charts/inventory`). **Debug**: Trace Firestore transactions with OpenTelemetry (deferred to Phase 5). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/inventory:1.0`, ~150MB). **Resource**: 512MB RAM (Kafka 512MB). |
| 2.3 Order | Build cart, checkout, and order tracking with substitutions and SKU-based discounts. | 2.5 weeks | 2.1, 2.2 | **Tech**: Django, PostgreSQL, `django-redis`, `requests`. **Models**: `Order(id: uuid, user_id: str, items: jsonb, status: str, substitution_prefs: jsonb, discount_id: fk, discount_applied: decimal, created_at: datetime)`, `Cart(user_id: str, items: jsonb, updated_at: datetime)`, `Discount(id: uuid, sku: str, percentage: decimal, valid_until: datetime, applies_to_expiring: bool)`. **API**: REST (`/cart`, `/cart/{user_id}`, `/checkout`, `/orders/{id}`, GET/POST, ≤1s). **Kafka**: Publish `OrderPlaced` with discount details to `order_events`. **Logic**: Validate stock via Inventory API, apply substitutions (e.g., `{sku: "milk", allow: true, alt_sku: "milk2"}`), fetch discounts (`SELECT * FROM discounts WHERE sku = :sku AND valid_until > now()`), apply highest percentage if `applies_to_expiring` and SKU expires <3 days, update `status`. **Caching**: Redis (`cart:{user_id}`, `discounts:{sku}`, 24-hour/1-hour TTL). **Rate Limiting**: NGINX (`limit_req zone=order burst=20`). **Circuit Breaker**: `tenacity` for Inventory API (retry 3x, timeout 500ms). **Health Check**: `/health`. **Scripts**: Mock checkout (`test_checkout.py`), simulate substitutions (`test_substitutions.py`), seed discounts (`seed_discounts.py`, e.g., 20% off near-expiry milk), test discounts (`test_discounts.py`). **Tests**: pytest for cart, checkout, substitutions, discounts (≥85% coverage); Testcontainers for PostgreSQL. **Metrics**: Prometheus for checkout latency, discount usage. **Logging**: Log checkout/discount errors to Sentry. **Grocery**: Support per-item substitutions, real-time stock validation, auto-apply near-expiry discounts. **Scalability**: Index `user_id`, `sku` on `Discount`. **HA**: 2 replicas in `kind` (`charts/order`). **Debug**: Log API call latencies (>800ms). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/order:1.0`, ~200MB). **Resource**: 512MB RAM. |
| 2.4 Payment | Integrate Stripe for secure, scalable payments with webhook handling, extensible for future providers. | 1 week | 2.3 | **Tech**: Leafjs, Stripe SDK (`@stripe/stripe-js`), PostgreSQL. **Model**: `Payment(id: uuid, order_id: uuid, stripe_id: str, amount: decimal, status: str, created_at: datetime)`. **API**: REST (`/pay`, POST, ≤1s; `/payment/{order_id}`, GET). **Kafka**: Publish `PaymentProcessed` to `payment_events`. **Logic**: Create Stripe PaymentIntent, handle webhooks (`payment_intent.succeeded`, `payment_intent.failed`) with idempotency. **Extensibility**: Implement `PaymentProcessor` interface (`processPayment`, `handleWebhook`) with Stripe; mock MoMo. **Security**: PCI DSS via Stripe.js, no card data stored. **Rate Limiting**: `nestjs-throttler` (50 req/s). **Circuit Breaker**: `opossum` for Stripe API (timeout 500ms). **Health Check**: `/health`. **Scripts**: Test webhook handler (`test_stripe.js`), simulate payments (`simulate_payment.js`). **Tests**: Jest for APIs, webhooks, interface compliance (≥85% coverage); mock Stripe with `stripe-mock`. **Metrics**: Prometheus for payment success rate, latency. **Logging**: Log webhook errors to Sentry. **Grocery**: Ensure fast payment processing. **Scalability**: Handle 300,000 peak users with async webhooks. **HA**: 2 replicas in `kind` (`charts/payment`). **Debug**: Log Stripe API errors. **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/payment:1.0`, ~150MB). **Resource**: 256MB RAM. |
| 2.5 User | Build JWT-based authentication, user profiles, and RBAC for secure access. | 1 week | 1.4 | **Tech**: Django, `djangorestframework-simplejwt`, PostgreSQL. **Model**: `User(id: uuid, email: str, password: str, role: str, name_en: str, name_vi: str, created_at: datetime)`. **API**: REST (`/login`, `/refresh`, `/profile`, `/users/{id}`, POST/GET). **Logic**: Issue JWT tokens (access: 15m, refresh: 24h), RBAC with roles (`customer`, `admin`, `staff`). **Security**: Hash passwords with `argon2`, enforce TLS 1.3. **Rate Limiting**: NGINX (`limit_req zone=user burst=100`). **Health Check**: `/health`. **Scripts**: Seed test users (`seed_users.py`), test RBAC (`test_rbac.py`). **Tests**: pytest for auth, RBAC, profile updates (≥85% coverage). **Metrics**: Prometheus for login success rate, token issuance. **Logging**: Log auth failures to Sentry. **Grocery**: Support multilingual names, role-based order access. **Scalability**: Index `email` for fast login. **HA**: 2 replicas in `kind` (`charts/user`). **Debug**: Log token validation errors. **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/user:1.0`, ~200MB). **Resource**: 256MB RAM. |
| 2.6 Delivery | Integrate Grab/ViettelPost for same-day delivery slots, in-store pickup, and real-time tracking, extensible for future providers. | 1.5 weeks | 2.3 | **Tech**: NestJS, Firestore, Redis, `nestjs-throttler`. **Model**: Firestore `delivery/{orderId}` with `{slot: timestamp, provider: str, status: str, tracking_url: str, pickup: bool}`. **API**: REST (`/slots`, `/delivery/{orderId}`, `/pickup`, GET/POST, ≤500ms). **Caching**: Redis (`slots:{date}`, 1-hour TTL). **Kafka**: Publish `DeliveryScheduled`, `DeliveryUpdated` to `delivery_events`. **Logic**: Mock Grab/ViettelPost APIs, fallback to next-day slots, support pickup locations. **Extensibility**: Implement `DeliveryProvider` interface (`fetchSlots`, `bookDelivery`, `trackDelivery`) with Grab/ViettelPost; mock Gojek. **Rate Limiting**: `nestjs-throttler` (200 req/s). **Circuit Breaker**: `opossum` for provider APIs (timeout 300ms). **Health Check**: `/health`. **Scripts**: Seed slots (`seed_slots.js`), test pickup (`test_pickup.js`). **Tests**: Jest for slot booking, cache hits, interface compliance (≥85% coverage). **Metrics**: Prometheus for slot booking success, latency. **Logging**: Log cache misses, API errors to Sentry. **Grocery**: Prioritize same-day slots, display tracking URLs. **Scalability**: Partition Firestore by `orderId`. **HA**: 2 replicas in `kind` (`charts/delivery`). **Debug**: Log provider API latency (>400ms). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/delivery:1.0`, ~150MB). **Resource**: 512MB RAM. |
| 2.7 Notification | Build scalable email/SMS notifications for order and delivery updates. | 1 week | 2.3, 2.6 | **Tech**: NestJS, Firestore, SendGrid/Twilio, `nestjs-throttler`. **Model**: Firestore `notifications/{id}` with `{order_id: str, type: str, status: str, sent_at: timestamp, payload: object}`. **API**: Internal (`/notify`, POST). **Kafka**: Consume `OrderConfirmed`, `DeliveryScheduled`, `PaymentProcessed`. **Logic**: Send multilingual email/SMS with substitution/discount details. **Rate Limiting**: `nestjs-throttler` (50 req/s). **Circuit Breaker**: `opossum` for SendGrid/Twilio APIs (timeout 500ms). **Health Check**: `/health`. **Scripts**: Test notifications (`test_notify.js`), simulate failures (`test_notify_fail.js`). **Tests**: Jest for message delivery, payload parsing (≥85% coverage); mock SendGrid/Twilio. **Metrics**: Prometheus for notification success rate. **Logging**: Log delivery failures to Sentry. **Grocery**: Include order status, substitutions, discounts in messages. **Scalability**: Batch notifications in Kafka consumer. **HA**: 2 replicas in `kind` (`charts/notification`). **Debug**: Log retry attempts. **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/notification:1.0`, ~150MB). **Resource**: 256MB RAM. |
| 2.8 Extensibility Prep | Design Payment and Delivery services with abstract interfaces for future MoMo/Gojek integrations. | 2 days | 2.4, 2.6 | **Tech**: NestJS, TypeScript. **Payment**: Implement `PaymentProcessor` interface (`processPayment`, `handleWebhook`) with Stripe; mock MoMo for testing. **Delivery**: Implement `DeliveryProvider` interface (`fetchSlots`, `bookDelivery`, `trackDelivery`) with Grab/ViettelPost; mock Gojek. **Config**: Store provider configs in Firestore (`providers/{id}`, e.g., `{name: "MoMo", apiKey: "..."}`). **Tests**: Jest for interface compliance (≥85% coverage); mock providers with `nock`. **Metrics**: Prometheus for provider switch latency. **Logging**: Log mock API calls to Sentry. **Grocery**: Ensure extensibility for Vietnam-specific providers. **Scalability**: Support dynamic provider selection. **HA**: N/A (config only). **CI/CD**: GitHub Actions. **Docker**: Included in Payment/Delivery images. **Resource**: 0MB (no new containers). |

**Grocery Focus**:
- **Inventory**: Prevent oversells with Firestore transactions, track perishable expiration, alert on low stock.
- **Order**: Support substitutions, validate stock in real-time, auto-apply near-expiry discounts.
- **Delivery**: Optimize same-day slots with Redis caching, provide tracking URLs, support pickup.
- **Product Catalog**: Offer dietary/expires_soon filters, display popular items for pseudo-personalization.
- **Notification**: Deliver multilingual messages with substitution/discount details.
- **Extensibility**: Prepare for MoMo/Gojek to support Vietnam market.
- **Performance**: Ensure ≤100ms inventory updates, ≤500ms slot responses, ≤1s checkout with caching.
- **Reliability**: Achieve ≥99% inventory accuracy, ≥98% slot booking success.

**Code Snippets**:
- **Product Catalog (API with Filters)**:
  ```python
  # services/product-catalog/product/views.py
  from rest_framework import viewsets
  from django.contrib.postgres.search import SearchVector
  from django_filters.rest_framework import DjangoFilterBackend
  from .models import Product
  from .serializers import ProductSerializer
  from django.utils import timezone
  from datetime import timedelta

  class ProductViewSet(viewsets.ModelViewSet):
      queryset = Product.objects.all()
      serializer_class = ProductSerializer
      filter_backends = [DjangoFilterBackend]
      filterset_fields = ['category', 'is_perishable', 'dietary', 'tags']

      def get_queryset(self):
          queryset = super().get_queryset()
          query = self.request.query_params.get('search')
          expires_soon = self.request.query_params.get('expires_soon')
          if query:
              queryset = queryset.annotate(
                  search=SearchVector('name_en', 'name_vi')
              ).filter(search=query)
          if expires_soon:
              queryset = queryset.filter(expires_at__lte=timezone.now() + timedelta(days=3))
          return queryset[:100]

  class PopularViewSet(viewsets.ReadOnlyModelViewSet):
      serializer_class = ProductSerializer

      def get_queryset(self):
          category = self.request.query_params.get('category')
          query = """
              SELECT p.*, COUNT(oi.sku) as sales
              FROM product_product p
              JOIN order_order_items oi ON p.sku = oi.sku
              WHERE oi.created_at > %s
              %s
              GROUP BY p.id
              ORDER BY sales DESC
              LIMIT 10
          """
          params = [timezone.now() - timedelta(days=7)]
          if category:
              query = query % ('%s', 'AND p.category_id = %s')
              params.append(category)
          else:
              query = query % ('%s', '')
          return Product.objects.raw(query, params)
  ```
- **Order (Discount Logic)**:
  ```python
  # services/order/order/models.py
  from django.db import models
  import uuid

  class Discount(models.Model):
      id = models.UUIDField(primary_key=True, default=uuid.uuid4)
      sku = models.CharField(max_length=50)
      percentage = models.DecimalField(max_digits=5, decimal_places=2)
      valid_until = models.DateTimeField()
      applies_to_expiring = models.BooleanField(default=False)

  class Order(models.Model):
      id = models.UUIDField(primary_key=True, default=uuid.uuid4)
      user_id = models.CharField(max_length=50)
      items = models.JSONField()
      status = models.CharField(max_length=20)
      substitution_prefs = models.JSONField(default=dict)
      discount = models.ForeignKey(Discount, null=True, on_delete=models.SET_NULL)
      discount_applied = models.DecimalField(max_digits=10, decimal_places=2, default=0)
      created_at = models.DateTimeField(auto_now_add=True)
  ```
  ```python
  # services/order/order/services.py
  import requests
  from django.db import transaction
  from django.utils import timezone
  from datetime import timedelta
  from .models import Order, Discount

  def process_checkout(user_id, items, substitution_prefs):
      with transaction.atomic():
          total_discount = 0
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
              product = requests.get(f'http://product-catalog:8000/products/{item["sku"]}').json()
              discounts = Discount.objects.filter(
                  sku=item['sku'],
                  valid_until__gt=timezone.now()
              )
              if product['is_perishable'] and product['expires_at'] < (timezone.now() + timedelta(days=3)):
                  discounts = discounts.filter(applies_to_expiring=True)
              discount = discounts.order_by('-percentage').first()
              if discount:
                  item_price = product['price'] * item['quantity']
                  item_discount = item_price * (discount.percentage / 100)
                  total_discount += item_discount
                  item['discount_id'] = str(discount.id)
          order = Order.objects.create(
              user_id=user_id,
              items=items,
              substitution_prefs=substitution_prefs,
              discount=discount,
              discount_applied=total_discount
          )
          requests.post('http://inventory:3000/stock/update', json=items)
          return order
  ```
- **Payment (Extensibility)**:
  ```typescript
  // services/payment/src/payment.processor.ts
  import { Injectable } from '@nestjs/common';

  interface PaymentProcessor {
    processPayment(orderId: string, amount: number): Promise<string>;
    handleWebhook(event: any): Promise<void>;
  }

  @Injectable()
  class StripeProcessor implements PaymentProcessor {
    async processPayment(orderId: string, amount: number): Promise<string> {
      const stripe = require('stripe')(process.env.STRIPE_KEY);
      const intent = await stripe.paymentIntents.create({
        amount: amount * 100,
        currency: 'vnd',
        metadata: { orderId },
      });
      return intent.id;
    }

    async handleWebhook(event: any): Promise<void> {
      // Handle Stripe webhook
    }
  }

  @Injectable()
  class MoMoProcessor implements PaymentProcessor {
    async processPayment(orderId: string, amount: number): Promise<string> {
      // Mock for testing
      return `momo_${orderId}`;
    }

    async handleWebhook(event: any): Promise<void> {
      // Mock for testing
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
      dietary = fake.random_element([[], ['vegan'], ['gluten-free'], ['vegan', 'gluten-free']])
      tags = fake.random_element([[], ['organic'], ['local']])
      Product.objects.create(
          sku=f"SKU{i:06d}",
          name_en=fake.word(),
          name_vi=fake.word(),
          category=fake.random_element(categories),
          price=fake.random_int(1, 1000) / 100,
          is_perishable=is_perishable,
          expires_at=fake.date_time_between(start_date='now', end_date='+30d') if is_perishable else None,
          dietary=dietary,
          tags=tags
      )
  ```
- **Seed Discounts**:
  ```python
  # scripts/seed_discounts.py
  from order.models import Discount
  from faker import Faker
  from django.utils import timezone
  fake = Faker()

  for i in range(100):
      Discount.objects.create(
          sku=f"SKU{fake.random_int(1, 50000):06d}",
          percentage=fake.random_int(10, 50) / 100,
          valid_until=fake.date_time_between(start_date='now', end_date='+30d'),
          applies_to_expiring=fake.random_element([True, False])
      )
  ```

**Resource Considerations**:
- **Laptop Specs**: Allocate ~8GB RAM for Docker containers (512MB each for Product Catalog, Inventory, Order, Delivery; 256MB for Payment, User, Notification; 512MB for Kafka; 2GB for dev container; 2GB for `kind`).
- **Optimization**: Run only active service containers (e.g., `docker stop payment`). Use `docker stats` to monitor CPU/RAM.
- **Kafka**: Single-broker Kafka (512MB); scale to 3 brokers in Phase 6.
- **Testing**: Run tests in dev container. Use `pytest --cov=product` and `jest --coverage`.
- **Constraints**: Ensure Docker supports ~10 containers. Use WSL2 (Windows) if needed.

**Output**: Fully functional MVP microservices with scalable REST APIs, Kafka-driven events, Redis caching, and grocery-specific features (perishable tracking, substitutions, same-day slots, discounts, filters, popular items). Docker images deployed to `kind` via Helm, integrated with CI/CD, tested for ≥85% coverage, ready for Phase 3.

### Phase 3: Frontend and UX (3–4 Weeks)
**Goal**: Develop a mobile-first frontend with grocery-specific UX (search, checkout, delivery slots).

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 3.1 Prototype | Design UX in Figma for search, cart, checkout, slots. | 1 week | None | Mobile-first, multilingual (English/Vietnamese). Single-page checkout, calendar slot picker. |
| 3.2 Build frontend | Develop React frontend, connect to GraphQL/REST APIs. | 2 weeks | 2.1–2.8, 3.1 | Use Apollo Client for GraphQL. Components: `ProductList`, `Cart`, `SlotPicker`. Cache with Apollo. |
| 3.3 Multilingual | Implement English/Vietnamese toggle. | 3 days | 3.2 | Use `react-i18next`. Fetch translations from Product Catalog API. |
| 3.4 Optimize | Ensure ≤2s page loads with Lighthouse. | 3 days | 3.2 | Compress images, use Cloudflare CDN, lazy-load products. |

**Grocery Focus**: Streamlined checkout with substitution/discount options, perishable/dietary filters, popular items, fast slot selection (≤500ms).
**Code Snippet (GraphQL Query)**:
```graphql
query GetProducts($category: String, $dietary: [String], $expiresSoon: Boolean) {
  products(category: $category, dietary: $dietary, expiresSoon: $expiresSoon) {
    sku
    nameEn
    nameVi
    price
    isPerishable
    dietary
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
| 4.3 Extensibility | Abstract Payment/Delivery Services for MoMo, Gojek. | 3 days | 2.8, 4.1, 4.2 | Strategy pattern in NestJS: `PaymentProcessor`, `DeliveryProvider`. Store configs in Vault. |

**Grocery Focus**: Ensure ≤1s payment processing, ≤500ms slot updates, reliable webhook handling.
**Output**: Fully integrated third-party services, extensible for future providers.

### Phase 5: Testing and Validation (3–4 Weeks)
**Goal**: Ensure reliability, performance, and grocery-specific functionality through comprehensive testing.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 5.1 Unit tests | Write tests for all services (≥80% coverage). | 1 week | 2.1–2.8 | pytest for Django, Jest for NestJS. Test discounts, filters, popular items. |
| 5.2 Integration tests | Test service interactions (REST, Kafka). | 1 week | 2.1–2.8, 4.1–4.3 | Testcontainers for PostgreSQL/Firestore. Mock Stripe/Grab APIs with `nock`. |
| 5.3 E2E tests | Test user flows (search, checkout, slots). | 1 week | 3.2, 5.2 | Cypress for browser tests. Test multilingual UX, substitutions, discounts. |
| 5.4 Load tests | Simulate 300,000 users, validate performance. | 3 days | 5.2 | Locust: Test ≤500ms search, ≤1s checkout. Monitor with Prometheus/Grafana. |
| 5.5 Security tests | Scan for vulnerabilities, test JWT/RBAC. | 2 days | 5.2 | OWASP ZAP for APIs, Snyk for dependencies. |

**Grocery Focus**: Test inventory accuracy (≥99%), slot booking success (≥98%), substitution/discount flows under load.
**Output**: Fully tested platform meeting performance and reliability goals.

### Phase 6: Deployment and Beta Launch (2–3 Weeks)
**Goal**: Deploy the MVP, conduct beta testing, and prepare for production.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 6.1 Staging deploy | Deploy to staging Kubernetes cluster. | 3 days | 2.1–2.8, 3.2, 4.1–4.3 | Use Helm: `helm upgrade grocery ./charts`. Blue-green for Inventory/Delivery. |
| 6.2 Beta testing | Launch to 100–500 users, collect feedback. | 1 week | 6.1, 5.1–5.5 | Use Django `waffle` for feature flags. Google Forms, Hotjar for feedback. |
| 6.3 Fix issues | Address beta feedback (e.g., UX, performance). | 1 week | 6.2 | Prioritize critical issues (e.g., slow checkout >1s). Update GitHub Issues. |
| 6.4 Production deploy | Deploy to production, monitor KPIs. | 3 days | 6.3 | Blue-green deployment, smoke tests (Postman). Monitor with Prometheus/Grafana. |

**Grocery Focus**: Validate real-time inventory (≤100ms), delivery slots (≤500ms), substitution/discount UX during beta.
**Output**: Production-ready MVP, initial KPIs (e.g., conversion rate ≥3%).

### Phase 7: Post-MVP Enhancements (6–8 Weeks, Optional)
**Goal**: Add deferred features and scale for growth.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 7.1 Promotions | Build advanced discount/coupon service. | 2 weeks | 2.3 | Django, PostgreSQL: `Promotion(id, type, discount, rules)`. API: `/promotions`. Support coupons, bulk discounts. |
| 7.2 Search | Implement advanced search with Elasticsearch. | 2 weeks | 2.1 | NestJS, Elasticsearch: Index 50,000 SKUs, support fuzzy search, dietary filters. |
| 7.3 Recommendation | Build personalized suggestions with collaborative filtering. | 2 weeks | 2.1, 2.3 | Django, PostgreSQL: `Recommendation(user_id, skus)`. Use matrix factorization. |
| 7.4 MoMo/Gojek | Integrate MoMo payments, Gojek delivery. | 2 weeks | 2.8, 4.3 | NestJS, REST APIs. Update Payment/Delivery abstractions. |

**Grocery Focus**: Advanced promotions for perishables, search filters for dietary needs, personalized recommendations, expanded delivery capacity.
**Output**: Enhanced platform ready for broader rollout.

## Timeline
- **Total Duration**: 26–35 weeks (6–9 months) for MVP (Phases 1–6), plus 6–8 weeks for post-MVP (Phase 7).
- **Breakdown**:
  - Phase 1: 1–2 weeks
  - Phase 2: 9–11 weeks
  - Phase 3: 3–4 weeks
  - Phase 4: 2–3 weeks
  - Phase 5: 3–4 weeks
  - Phase 6: 2–3 weeks
  - Phase 7: 6–8 weeks (optional)
- **Assumption**: 20–30 hours/week, with flexibility for adjustments.

## Best Practices for Solo Development
- **Prioritization**: Focus on MVP (Product Catalog, Inventory, Order, Payment, User, Delivery, Notification). Treat discounts, filters, popular items as secondary.
- **Automation**:
  - Use GitHub Actions for CI/CD (lint, test, deploy).
  - Deploy monitoring (Prometheus/Grafana, Loki) early.
  - Script tasks (e.g., `seed_discounts.py`, `seed_products.py`) in `/scripts`.
- **Task Management**: Use GitHub Projects with Kanban board. Label tasks (e.g., `priority:high`, `grocery`, `enhancement`) and set milestones (“MVP”).
- **Technical Debt**:
  - Allocate 10% of time to refactoring (e.g., optimize discount queries).
  - Use Snyk to catch vulnerabilities.
- **Grocery-Specific**:
  - Test inventory locking, substitutions, discounts rigorously.
  - Cache delivery slots (Redis) for ≤500ms responses.
  - Validate multilingual UX in beta.
- **Cloud Prep**: Use cloud-agnostic tools (MinIO, Helm) and test staging for AWS/GCP.

## Monitoring and Validation
- **KPIs** (Phase 6):
  - Conversion Rate: ≥3%.
  - Order Fulfillment: ≥95%.
  - Inventory Accuracy: ≥99%.
  - Slot Booking Success: ≥98%.
  - Uptime: ≥99.9%.
- **Tools**: Prometheus/Grafana for metrics, GA4 for behavior, Hotjar for UX, Firestore for events.
- **Feedback**: Collect beta feedback via Google Forms/Hotjar to refine UX.

## Risks and Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| Overrunning timeline | Delays launch | Prioritize MVP, automate tests, use GitHub Projects. |
| Performance issues | Slow UX (>1s checkout) | Cache with Redis, test with Locust, monitor with Prometheus. |
| Third-party failures | Payment/delivery disruptions | Use circuit breakers, mock APIs, cache responses. |
| Technical debt | Maintenance burden | Refactor incrementally, enforce ≥80% coverage, use Snyk. |
| Feature creep | Delayed Phase 2 | Limit discounts to SKU-based, filters to predefined categories, recommendations to static lists. |

## Next Steps
- Start Phase 1 by initializing repo and Kubernetes cluster.
- Review tasks in GitHub Projects weekly, adjusting priorities.
- Deploy monitoring early (Phase 1) to track progress.
- Revisit plan post-MVP to prioritize post-MVP features (e.g., advanced promotions, MoMo).
