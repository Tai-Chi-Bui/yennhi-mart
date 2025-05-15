# Development Plan

## Overview
This plan outlines the development of a large-scale, microservices-based grocery e-commerce platform supporting 100,000 daily active users (DAU), 300,000 peak users, and 50,000 SKUs. The platform prioritizes scalability, performance (≤500ms search, ≤2s page load, ≤100ms inventory updates, 99.9% uptime), and grocery-specific features (e.g., real-time inventory, same-day delivery, substitutions, multilingual English/Vietnamese UX, promotions, advanced search, personalized recommendations). It runs on Kubernetes/Docker on-premises, with future cloud migration (e.g., AWS EKS). As a solo developer, the plan focuses on automation, prioritization, and modularity to streamline implementation over 25–34 weeks (6–8 months) at 20–30 hours/week.

## Scope
- **Features**:
  - Product Catalog: Browse/search 50,000 SKUs, multilingual descriptions, dietary/expiry filters, popular items, promotions.
  - Inventory: Real-time stock updates (≤100ms), perishable tracking, substitutions.
  - Order: Cart, checkout (≤1s), order tracking, discounts, coupons.
  - Payment: Stripe and MoMo integration, PCI DSS compliant.
  - User: JWT-based authentication, basic profiles.
  - Delivery: Grab, ViettelPost, Gojek integration, same-day slots (≤500ms), in-store pickup.
  - Notifications: Order/delivery updates (email/SMS).
  - Search: Elasticsearch-based advanced search with fuzzy matching, dietary filters.
  - Recommendation: Personalized suggestions with rule-based and collaborative filtering.
- **Non-Functional**:
  - Scalability: Handle 300,000 peak users.
  - Performance: ≤500ms search, ≤2s page load, ≤100ms inventory updates.
  - Uptime: ≥99.9%.
  - Security: JWT, RBAC, TLS 1.3, PCI DSS compliance.

## Tech Stack
- **Backend**: Python (Django) for transactional services, Node.js (NestJS) for real-time services.
- **Databases**: PostgreSQL (transactional), Firestore (real-time), Elasticsearch (search).
- **Infrastructure**: Kubernetes/Docker, NGINX Ingress, Redis (caching), Cloudflare (CDN).
- **Messaging**: Kafka for async events.
- **CI/CD**: GitHub Actions with self-hosted runners.
- **Monitoring**: Prometheus/Grafana, Loki, OpenTelemetry/Jaeger.
- **Testing**: pytest, Jest, Cypress, Locust, Testcontainers.
- **Third-Party**: Stripe/MoMo (payments), Grab/ViettelPost/Gojek (delivery), SendGrid/Twilio (notifications).

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

### Phase 2: Core Microservices and Feature Development (14–16 Weeks)
**Goal**: Build a comprehensive set of microservices (Product Catalog, Inventory, Order, Payment, User, Delivery, Notification, Search, Recommendation) with all features (promotions, advanced search, personalized recommendations, MoMo/Gojek integrations) for a large-scale grocery e-commerce platform. Ensure scalability, performance, and grocery-specific functionality (e.g., perishable tracking, substitutions, same-day delivery, dietary search, promotions). Each service includes APIs, database models, Kafka event handling, caching, rate limiting, circuit breakers, health checks, and automated tests, leveraging Phase 1’s Docker environment. Optimized for a solo developer to support 100,000 DAU and 300,000 peak users.

**Development Environment Setup**:
- Use the following `docker-compose.yml` to run essential services (PostgreSQL, Firestore emulator, Elasticsearch, Kafka, Redis) with memory limits for your laptop (8–12GB RAM, 4 cores):
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
    elasticsearch:
      image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
      environment:
        - discovery.type=single-node
        - xpack.security.enabled=false
        - "ES_JAVA_OPTS=-Xms512m -Xmx1g"
      ports:
        - "9200:9200"
      deploy:
        resources:
          limits:
            memory: 1g
    kafka:
      image: bitnami/kafka:3.7
      ports:
        - "9092:9092"
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
- **Setup Instructions**:
  1. Save as `docker-compose.yml`.
  2. Run `docker-compose up -d`.
  3. Monitor with `docker stats` and stop unused containers (`docker stop <name>`) to free RAM.
  4. For each microservice, create a separate `Dockerfile` or `docker-compose.yml`, linking to required services (e.g., PostgreSQL for Product Catalog).

**CI/CD Pipeline**:
- Use GitHub Actions for linting, testing, building, and deploying Docker images.
- Example workflow:
  ```yaml
  name: CI
  on: [push]
  jobs:
    build:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v3
        - name: Set up Docker Buildx
          uses: docker/setup-buildx-action@v2
        - name: Lint
          run: flake8 . && eslint .
        - name: Test
          run: pytest && jest
        - name: Build Docker Image
          run: docker build -t grocery/$SERVICE:test .
  ```

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 2.1 Product Catalog | Build a scalable service for 50,000 SKUs with multilingual descriptions, basic search, dietary/expiry filters, popular items, and promotion integration. | 2.5 weeks | 1.4 | **Tech**: Django 4.2, PostgreSQL, `django-environ`, `django-filter`, `django-rest-framework`. **Models**: `Product(sku: str, name_en: str, name_vi: str, category_id: fk, price: decimal, is_perishable: bool, expires_at: datetime, stock_updated_at: datetime, dietary: array[str], tags: array[str], unit_of_measure: str, brand: str, manufacturer: str, image_urls: array[url])`, `Category(id: uuid, name_en: str, name_vi: str)`, `Promotion(id: uuid, type: str, percentage: decimal, valid_until: datetime, rules: jsonb, min_order: decimal)`. **API**: REST (`/products`, `/products/{sku}`, `/categories`, `/products/popular`, `/promotions`, `/products/promoted`, `/products/{sku}/images`, GET/POST/PUT) with pagination, filtering (`category`, `is_perishable`, `dietary`, `expires_soon`, `tags`, `price_range`). **Search**: PostgreSQL GIN index on `name_en`, `name_vi`, `dietary`, `tags` for ≤500ms queries; sync with Elasticsearch (2.8). **Elasticsearch Mapping**: Fields: `sku`, `name_en`, `name_vi`, `category`, `price`, `dietary`, `tags`, `is_perishable`, `expires_at`; custom `vi_analyzer` for Vietnamese. **Popular Items**: `SELECT sku, COUNT(*) FROM order_items WHERE created_at > now() - interval '7 days' GROUP BY sku ORDER BY COUNT DESC LIMIT 10`. **Promotions**: Support SKU-based, category-based, perishable-specific, buy-one-get-one, bulk discounts (`rules: {"sku": "milk", "expires_days": 3, "min_quantity": 2}`). **Caching**: Redis (`products:{sku}`, `popular:{category}`, `promotions:{id}`, 1-hour TTL). **Rate Limiting**: NGINX (`limit_req zone=products burst=50`). **Circuit Breaker**: `tenacity` for Elasticsearch (retry 3x, timeout 500ms). **Health Check**: `/health`. **Kafka**: Publish `ProductUpdated`, `PromotionUpdated` to `product_events`. **Scripts**: Seed 50,000 SKUs (`seed_products.py`, 20% perishables, 10% vegan/gluten-free), promotions (`seed_promotions.py`). **Tests**: pytest for filters, popular items, promotions (≥85% coverage); Testcontainers for PostgreSQL. **Metrics**: Prometheus for filter usage, cache hits, promotion application. **Logging**: Log slow queries (>300ms) to Sentry. **Grocery**: Support dietary filters, near-expiry highlights, popular items, perishable promotions. **Scalability**: Shard PostgreSQL by `category_id`; index `dietary`, `tags`. **HA**: 2 replicas in `kind` via Helm (`charts/product-catalog`). **Debug**: Use Django Debug Toolbar locally. **CI/CD**: GitHub Actions (`lint`, `test`, `build`, `push`). **Docker**: Image (`grocery/product-catalog:1.0`, ~200MB). **Resource**: 512MB RAM. |
| 2.2 Inventory | Build a real-time stock management service with perishable tracking, optimistic locking, and Kafka integration. | 2 weeks | 1.4, 1.5 | **Tech**: NestJS 10, Firestore emulator, Kafka (`bitnami/kafka:3.7`), `nestjs-throttler`. **Model**: Firestore `inventory/{location}/{sku}` with `{stock: number, reserved_stock: number, expires_at: timestamp, last_updated: timestamp, low_stock_threshold: number, location: str}`. **API**: REST (`/stock/{sku}`, `/stock/batch`, `/stock/{sku}/reserve`, `/stock/{sku}/release`, GET/POST, ≤100ms). **Kafka**: Deploy single-broker Kafka in Docker (`docker run -d --memory=512m bitnami/kafka`), topics: `order_events`, `product_events`. Consume `OrderPlaced` to deduct stock, `ProductUpdated` to sync SKUs. Publish `StockUpdated` on low stock. **Logic**: Optimistic locking via Firestore transactions; alert on low stock (<`low_stock_threshold`). **Caching**: Redis (`stock:{sku}`, 30s TTL). **Rate Limiting**: `nestjs-throttler` (100 req/s). **Circuit Breaker**: `opossum` for Firestore calls (timeout 200ms). **Health Check**: `/health` (Firestore, Kafka, Redis). **Scripts**: Seed 50,000 SKUs (`seed_inventory.js`), simulate stock updates (`test_stock.js`). **Tests**: Jest for APIs, transactions, Kafka consumers (≥85% coverage); mock Kafka with `kafkajs`, Firestore with emulator. **Metrics**: Prometheus (`prom-client`) for stock update latency, error rates. **Logging**: Winston to Loki (deferred to Phase 5), console locally. **Grocery**: Track expiration, support multi-location stock. **Scalability**: Partition Firestore by `location`. **HA**: 2 replicas in `kind` (`charts/inventory`). **Debug**: Trace Firestore transactions with OpenTelemetry (deferred to Phase 5). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/inventory:1.0`, ~150MB). **Resource**: 512MB RAM (Kafka 512MB). |
| 2.3 Order | Build cart, checkout, and order tracking with substitutions, discounts, and coupons. | 3 weeks | 2.1, 2.2 | **Tech**: Django, PostgreSQL, `django-redis`, `requests`. **Models**: `Order(id: uuid, user_id: str, items: jsonb, status: str, substitution_prefs: jsonb, promotion_id: fk, discount_applied: decimal, coupon_code: str, created_at: datetime, delivery_instructions: text)`, `Cart(user_id: str, items: jsonb, updated_at: datetime)`. **API**: REST (`/cart`, `/cart/{user_id}`, `/checkout`, `/orders/{id}`, `/orders/{id}/apply-coupon`, `/orders/{id}/confirm-substitutions`, GET/POST, ≤1s). **Kafka**: Publish `OrderPlaced` with promotion/coupon details to `order_events`. Consume `PaymentProcessed` to update status. **Logic**: Validate stock via Inventory API, apply substitutions (e.g., `{sku: "milk", allow: true, alt_sku: "milk2"}`), fetch promotions from Product Catalog (`/promotions`), apply highest discount (SKU-based, category-based, or coupon) if valid (`min_order` met, `valid_until` > now), update `status` (`pending`, `confirmed`, `fulfilled`). **Caching**: Redis (`cart:{user_id}`, `promotions:{id}`, `coupons:{code}`, 24-hour/1-hour TTL). **Rate Limiting**: NGINX (`limit_req zone=order burst=20`). **Circuit Breaker**: `tenacity` for Inventory/Product Catalog APIs (retry 3x, timeout 500ms). **Health Check**: `/health` (DB, Redis, Inventory, Product Catalog). **Scripts**: Mock checkout (`test_checkout.py`), simulate substitutions (`test_substitutions.py`), seed promotions/coupons (`seed_promotions.py`), test promotions (`test_promotions.py`). **Tests**: pytest for cart, checkout, substitutions, promotions, coupons (≥85% coverage); Testcontainers for PostgreSQL. **Metrics**: Prometheus for checkout latency, promotion usage, coupon application. **Logging**: Log checkout/promotion/coupon errors to Sentry. **Grocery**: Support per-item substitutions, real-time stock validation, auto-apply perishable discounts/coupons. **Scalability**: Index `user_id`, `promotion_id`, `coupon_code`. **HA**: 2 replicas in `kind` (`charts/order`). **Debug**: Log API call latencies (>800ms). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/order:1.0`, ~200MB). **Resource**: 512MB RAM. |
| 2.4 Payment | Integrate Stripe and MoMo for secure, scalable payments with webhook handling. | 2 weeks | 2.3 | **Tech**: NestJS, Stripe SDK (`@stripe/stripe-js`), MoMo SDK, PostgreSQL. **Model**: `Payment(id: uuid, order_id: uuid, provider: str, provider_id: str, amount: decimal, status: str, created_at: datetime)`. **API**: REST (`/pay`, `/payment/{order_id}`, `/payments/{id}/refund`, POST/GET, ≤1s). **Kafka**: Publish `PaymentProcessed` to `payment_events`. **Logic**: Implement `PaymentProcessor` interface (`processPayment`, `handleWebhook`) for Stripe and MoMo; create PaymentIntent for Stripe, equivalent for MoMo; handle webhooks (`payment_intent.succeeded`, `payment_intent.failed` for Stripe; MoMo equivalents) with idempotency. **Security**: PCI DSS via Stripe.js/MoMo SDK, no card data stored. **Rate Limiting**: `nestjs-throttler` (50 req/s). **Circuit Breaker**: `opossum` for provider APIs (timeout 500ms). **Health Check**: `/health` (DB, Stripe, MoMo). **Scripts**: Test webhooks (`test_payment.js`), simulate payments (`simulate_payment.js`). **Tests**: Jest for APIs, webhooks, provider switching (≥85% coverage); mock Stripe with `stripe-mock`, MoMo with `nock`. **Metrics**: Prometheus for payment success rate, latency, provider usage. **Logging**: Log webhook errors to Sentry. **Grocery**: Ensure fast payment processing. **Scalability**: Async webhooks. **HA**: 2 replicas in `kind` (`charts/payment`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/payment:1.0`, ~150MB). **Resource**: 256MB RAM. |
| 2.5 User | Build JWT-based authentication, user profiles, and RBAC for secure access. | 1 week | 1.4 | **Tech**: Django, `djangorestframework-simplejwt`, PostgreSQL. **Model**: `User(id: uuid, email: str, password: str, role: str, name_en: str, name_vi: str, dietary_preferences: array[str], favorite_products: array[str], addresses: jsonb)`. **API**: REST (`/login`, `/refresh`, `/profile`, `/users/{id}`, POST/GET). **Logic**: Issue JWT tokens (access: 15m, refresh: 24h), RBAC with roles (`customer`, `admin`, `staff`). **Security**: Hash passwords with `argon2`, enforce TLS 1.3. **Rate Limiting**: NGINX (`limit_req zone=user burst=100`). **Health Check**: `/health`. **Scripts**: Seed test users (`seed_users.py`), test RBAC (`test_rbac.py`). **Tests**: pytest for auth, RBAC, profile updates (≥85% coverage). **Metrics**: Prometheus for login success rate, token issuance. **Logging**: Log auth failures to Sentry. **Grocery**: Support multilingual names, role-based order access, dietary preferences. **Scalability**: Index `email` for fast login. **HA**: 2 replicas in `kind` (`charts/user`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/user:1.0`, ~200MB). **Resource**: 256MB RAM. |
| 2.6 Delivery | Integrate Grab, ViettelPost, and Gojek for same-day delivery slots, in-store pickup, and real-time tracking. | 2.5 weeks | 2.3 | **Tech**: NestJS, Firestore, Redis, `nestjs-throttler`. **Model**: Firestore `delivery/{orderId}` with `{slot: timestamp, provider: str, status: str, tracking_url: str, pickup: bool}`. **API**: REST (`/slots`, `/delivery/{orderId}`, `/pickup`, GET/POST, ≤500ms). **Caching**: Redis (`slots:{date}:{provider}`, 1-hour TTL). **Kafka**: Publish `DeliveryScheduled`, `DeliveryUpdated` to `delivery_events`. **Logic**: Implement `DeliveryProvider` interface (`fetchSlots`, `bookDelivery`, `trackDelivery`) for Grab, ViettelPost, Gojek; mock APIs initially, fallback to next-day slots, support pickup locations. **Rate Limiting**: `nestjs-throttler` (200 req/s). **Circuit Breaker**: `opossum` for provider APIs (timeout 300ms). **Health Check**: `/health`. **Scripts**: Seed slots (`seed_slots.js`), test pickup (`test_pickup.js`), test provider switching (`test_delivery_providers.js`). **Tests**: Jest for slot booking, cache hits, provider switching (≥85% coverage). **Metrics**: Prometheus for slot booking success, latency, provider usage. **Logging**: Log cache misses, API errors to Sentry. **Grocery**: Prioritize same-day slots, display tracking URLs. **Scalability**: Partition Firestore by `orderId`. **HA**: 2 replicas in `kind` (`charts/delivery`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/delivery:1.0`, ~150MB). **Resource**: 512MB RAM. |
| 2.7 Notification | Build scalable email/SMS notifications for order and delivery updates. | 1 week | 2.3, 2.6 | **Tech**: NestJS, Firestore, SendGrid/Twilio, `nestjs-throttler`. **Model**: Firestore `notifications/{id}` with `{order_id: str, type: str, status: str, sent_at: timestamp, payload: object}`. **API**: Internal (`/notify`, POST). **Kafka**: Consume `OrderConfirmed`, `DeliveryScheduled`, `PaymentProcessed`. **Logic**: Send multilingual email/SMS with substitution, promotion, delivery details. **Rate Limiting**: `nestjs-throttler` (50 req/s). **Circuit Breaker**: `opossum` for SendGrid/Twilio APIs (timeout 500ms). **Health Check**: `/health`. **Scripts**: Test notifications (`test_notify.js`), simulate failures (`test_notify_fail.js`). **Tests**: Jest for message delivery, payload parsing (≥85% coverage); mock SendGrid/Twilio. **Metrics**: Prometheus for notification success rate. **Logging**: Log delivery failures to Sentry. **Grocery**: Include order status, substitutions, promotions in messages. **Scalability**: Batch notifications in Kafka consumer. **HA**: 2 replicas in `kind` (`charts/notification`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/notification:1.0`, ~150MB). **Resource**: 256MB RAM. |
| 2.8 Search | Implement advanced search with Elasticsearch for 50,000 SKUs. | 2 weeks | 2.1 | **Tech**: NestJS, Elasticsearch (`docker.elastic.co/elasticsearch:8.10.0`), `nestjs-throttler`. **Setup**: Deploy Elasticsearch in Docker (`docker run -d --memory=1g elasticsearch:8.10.0`). **Index**: `products` index with fields (`sku`, `name_en`, `name_vi`, `category`, `price`, `dietary`, `tags`, `is_perishable`, `expires_at`). **API**: REST (`/search`, GET, ≤500ms) with fuzzy search, filters (`dietary`, `price_range`, `expires_soon`). **Logic**: Sync PostgreSQL products to Elasticsearch via Kafka (`ProductUpdated`); handle partial matches (e.g., “mil” → “milk”). **Caching**: Redis (`search:{query}`, 1-hour TTL). **Rate Limiting**: `nestjs-throttler` (100 req/s). **Health Check**: `/health`. **Scripts**: Seed index (`seed_elasticsearch.js`), test fuzzy search (`test_search.js`). **Tests**: Jest for search queries, sync logic (≥85% coverage). **Metrics**: Prometheus for search latency, hit rate. **Logging**: Log search errors to Sentry. **Grocery**: Support dietary/perishable filters, fuzzy search for Vietnamese terms. **Scalability**: Single-node Elasticsearch; scale to cluster post-MVP. **HA**: 1 replica in `kind` (`charts/search`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/search:1.0`, ~150MB). **Resource**: 1GB RAM (Elasticsearch). |
| 2.9 Recommendation | Build personalized suggestions with rule-based and collaborative filtering. | 2 weeks | 2.1, 2.3 | **Tech**: Django, PostgreSQL, `django-environ`. **Model**: `Recommendation(id: uuid, user_id: str, skus: array[str], created_at: datetime)`. **API**: REST (`/recommendations/{user_id}`, GET, ≤500ms). **Logic**: Rule-based (suggest SKUs from user’s category preferences) + collaborative filtering (item-item similarity: `SELECT sku2, COUNT(*) FROM order_items oi1 JOIN order_items oi2 ON oi1.order_id = oi2.order_id WHERE oi1.sku = :sku GROUP BY sku2`). **Caching**: Redis (`recommendations:{user_id}`, 1-hour TTL). **Rate Limiting**: NGINX (`limit_req zone=recommendations burst=50`). **Health Check**: `/health`. **Scripts**: Seed recommendations (`seed_recommendations.py`), test suggestions (`test_recommendations.py`). **Tests**: pytest for rules, collaborative filtering (≥85% coverage). **Metrics**: Prometheus for recommendation latency, click-through rate. **Logging**: Log errors to Sentry. **Grocery**: Suggest perishables, dietary-compatible items. **Scalability**: Index `user_id`, `sku`. **HA**: 2 replicas in `kind` (`charts/recommendation`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/recommendation:1.0`, ~200MB). **Resource**: 512MB RAM. |

**Grocery Focus**:
- **Inventory**: Prevent oversells with Firestore transactions, track perishable expiration, alert on low stock.
- **Order**: Support substitutions, validate stock, apply discounts/coupons (e.g., perishable-specific deals).
- **Delivery**: Optimize same-day slots, support multiple providers (Grab, ViettelPost, Gojek).
- **Product Catalog**: Offer dietary/expires_soon filters, popular items, promotion management.
- **Search**: Enable fuzzy search, dietary/perishable filters.
- **Recommendation**: Suggest perishables, dietary-compatible items.
- **Payment**: Support Stripe/MoMo for Vietnam market.
- **Notification**: Deliver multilingual messages with order, substitution, promotion details.
- **Performance**: Ensure ≤100ms inventory updates, ≤500ms search/slots, ≤1s checkout.
- **Reliability**: Achieve ≥99% inventory accuracy, ≥98% slot booking success, ≥95% promotion accuracy.

**Code Snippets**:
- **Product Catalog (API with Promotions and Filters)**:
  ```python
  # services/product-catalog/product/models.py
  from django.contrib.postgres.fields import ArrayField
  from django.db import models
  import uuid

  class Product(models.Model):
      sku = models.CharField(max_length=50, unique=True)
      name_en = models.CharField(max_length=255)
      name_vi = models.CharField(max_length=255)
      category = models.ForeignKey('Category', on_delete=models.CASCADE)
      price = models.DecimalField(max_digits=10, decimal_places=2)
      is_perishable = models.BooleanField(default=False)
      expires_at = models.DateTimeField(null=True, blank=True)
      stock_updated_at = models.DateTimeField(auto_now=True)
      dietary = ArrayField(models.CharField(max_length=50), default=list)
      tags = ArrayField(models.CharField(max_length=50), default=list)
      unit_of_measure = models.CharField(max_length=20, choices=[('kg', 'Kilogram'), ('liter', 'Liter'), ('piece', 'Piece')])
      brand = models.CharField(max_length=100, blank=True)
      manufacturer = models.CharField(max_length=100, blank=True)
      image_urls = ArrayField(models.URLField(), default=list)

  class Category(models.Model):
      id = models.UUIDField(primary_key=True, default=uuid.uuid4)
      name_en = models.CharField(max_length=255)
      name_vi = models.CharField(max_length=255)

  class Promotion(models.Model):
      id = models.UUIDField(primary_key=True, default=uuid.uuid4)
      type = models.CharField(max_length=20, choices=[
          ('sku', 'SKU'), ('category', 'Category'), ('perishable', 'Perishable'),
          ('buy_one_get_one', 'Buy One Get One'), ('bulk_discount', 'Bulk Discount')
      ])
      percentage = models.DecimalField(max_digits=5, decimal_places=2)
      valid_until = models.DateTimeField()
      rules = models.JSONField(default=dict)
      min_order = models.DecimalField(max_digits=10, decimal_places=2, default=0)
  ```
  ```python
  # services/product-catalog/product/views.py
  from rest_framework import viewsets
  from django.contrib.postgres.search import SearchVector
  from django_filters.rest_framework import DjangoFilterBackend
  from django.utils import timezone
  from datetime import timedelta
  from .models import Product, Promotion
  from .serializers import ProductSerializer, PromotionSerializer

  class ProductViewSet(viewsets.ModelViewSet):
      queryset = Product.objects.all()
      serializer_class = ProductSerializer
      filter_backends = [DjangoFilterBackend]
      filterset_fields = ['category', 'is_perishable', 'dietary', 'tags']

      def get_queryset(self):
          queryset = super().get_queryset()
          query = self.request.query_params.get('search')
          expires_soon = self.request.query_params.get('expires_soon')
          price_range = self.request.query_params.get('price_range')
          if query:
              queryset = queryset.annotate(
                  search=SearchVector('name_en', 'name_vi')
              ).filter(search=query)
          if expires_soon:
              queryset = queryset.filter(expires_at__lte=timezone.now() + timedelta(days=3))
          if price_range:
              min_price, max_price = map(float, price_range.split(','))
              queryset = queryset.filter(price__gte=min_price, price__lte=max_price)
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

  class PromotionViewSet(viewsets.ModelViewSet):
      queryset = Promotion.objects.all()
      serializer_class = PromotionSerializer
  ```
- **Inventory (Firestore Transaction)**:
  ```typescript
  // services/inventory/src/inventory.service.ts
  import { getFirestore, runTransaction, doc } from 'firebase/firestore';

  async function updateStock(sku: string, location: string, quantity: number) {
    const db = getFirestore();
    const inventoryRef = doc(db, `inventory/${location}/${sku}`);
    try {
      await runTransaction(db, async (transaction) => {
        const inventoryDoc = await transaction.get(inventoryRef);
        if (!inventoryDoc.exists()) {
          throw new Error('Document does not exist!');
        }
        const currentStock = inventoryDoc.data().stock;
        const newStock = currentStock - quantity;
        if (newStock < 0) {
          throw new Error('Not enough stock!');
        }
        transaction.update(inventoryRef, { stock: newStock });
      });
    } catch (error) {
      throw new Error(`Stock update failed: ${error.message}`);
    }
  }
  ```
- **Order (Checkout with Promotions)**:
  ```python
  # services/order/order/services.py
  import requests
  from django.db import transaction
  from django.utils import timezone
  from datetime import timedelta
  from .models import Order

  def process_checkout(user_id, items, substitution_prefs, coupon_code=None):
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
              promotions = requests.get(f'http://product-catalog:8000/promotions', params={
                  'sku': item['sku'],
                  'category': product['category_id'],
                  'is_perishable': product['is_perishable']
              }).json()
              applicable_promotion = None
              max_discount = 0
              for promo in promotions:
                  if promo['valid_until'] > timezone.now().isoformat():
                      if promo['type'] == 'sku' and promo['rules']['sku'] == item['sku']:
                          discount = product['price'] * item['quantity'] * (promo['percentage'] / 100)
                          if discount > max_discount:
                              max_discount = discount
                              applicable_promotion = promo
                      elif promo['type'] == 'perishable' and product['is_perishable'] and \
                           product['expires_at'] < (timezone.now() + timedelta(days=promo['rules']['expires_days'])).isoformat():
                          discount = product['price'] * item['quantity'] * (promo['percentage'] / 100)
                          if discount > max_discount:
                              max_discount = discount
                              applicable_promotion = promo
              total_discount += max_discount
              item['promotion_id'] = applicable_promotion['id'] if applicable_promotion else None
          if coupon_code:
              coupon = requests.get(f'http://product-catalog:8000/promotions', params={'code': coupon_code}).json()
              if coupon and coupon[0]['valid_until'] > timezone.now().isoformat():
                  total = sum(item['price'] * item['quantity'] for item in items)
                  if total >= coupon[0]['min_order']:
                      coupon_discount = total * (coupon[0]['percentage'] / 100)
                      total_discount += coupon_discount
          order = Order.objects.create(
              user_id=user_id,
              items=items,
              substitution_prefs=substitution_prefs,
              promotion_id=applicable_promotion['id'] if applicable_promotion else None,
              discount_applied=total_discount,
              coupon_code=coupon_code
          )
          requests.post('http://inventory:3000/stock/update', json=items)
          return order
  ```
- **Search (Elasticsearch Integration)**:
  ```typescript
  // services/search/src/search.controller.ts
  import { Controller, Get, Query } from '@nestjs/common';
  import { ElasticsearchService } from '@nestjs/elasticsearch';

  @Controller('search')
  export class SearchController {
    constructor(private readonly esService: ElasticsearchService) {}

    @Get()
    async search(
      @Query('q') query: string,
      @Query('dietary') dietary: string,
      @Query('price_range') priceRange: string,
      @Query('expires_soon') expiresSoon: string,
    ) {
      const body: any = {
        query: {
          bool: {
            must: query ? { multi_match: { query, fields: ['name_en', 'name_vi'], fuzziness: 'AUTO' } } : [],
            filter: [],
          },
        },
      };
      if (diet wagen: true
      if (dietary) body.query.bool.filter.push({ term: { dietary } });
      if (priceRange) {
        const [min, max] = priceRange.split(',').map(Number);
        body.query.bool.filter.push({ range: { price: { gte: min, lte: max } } });
      }
      if (expiresSoon) {
        body.query.bool.filter.push({
          range: { expires_at: { lte: new Date(Date.now() + 3 * 24 * 60 * 60 * 1000).toISOString() } },
        });
      }
      const { hits } = await this.esService.search({ index: 'products', body });
      return hits.hits.map(hit => hit._source);
    }
  }
  ```
- **Recommendation (Collaborative Filtering)**:
  ```python
  # services/recommendation/recommendation/views.py
  from rest_framework import viewsets
  from django.db import connection
  from .models import Recommendation
  from .serializers import RecommendationSerializer

  class RecommendationViewSet(viewsets.ReadOnlyModelViewSet):
      serializer_class = RecommendationSerializer

      def get_queryset(self):
          user_id = self.kwargs['user_id']
          with connection.cursor() as cursor:
              cursor.execute("""
                  SELECT p.sku
                  FROM product_product p
                  JOIN order_order o ON o.user_id = %s
                  JOIN order_order_items oi ON oi.order_id = o.id
                  WHERE p.category_id = (
                      SELECT category_id
                      FROM order_order_items oi2
                      JOIN order_order o2 ON o2.id = oi2.order_id
                      WHERE o2.user_id = %s
                      GROUP BY category_id
                      ORDER BY COUNT(*) DESC
                      LIMIT 1
                  )
                  LIMIT 5
              """, [user_id, user_id])
              rule_based = [row[0] for row in cursor.fetchall()]
              cursor.execute("""
                  SELECT oi2.sku, COUNT(*) as co_purchases
                  FROM order_order_items oi1
                  JOIN order_order_items oi2 ON oi1.order_id = oi2.order_id
                  JOIN order_order o ON o.id = oi1.order_id
                  WHERE o.user_id = %s AND oi1.sku != oi2.sku
                  GROUP BY oi2.sku
                  ORDER BY co_purchases DESC
                  LIMIT 5
              """, [user_id])
              collaborative = [row[0] for row in cursor.fetchall()]
          skus = list(set(rule_based + collaborative))
          return Recommendation.objects.create(user_id=user_id, skus=skus)
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
- **Seed Promotions**:
  ```python
  # scripts/seed_promotions.py
  from product.models import Promotion
  from faker import Faker
  from django.utils import timezone
  fake = Faker()

  for i in range(100):
      promotion_type = fake.random_element(['sku', 'category', 'perishable'])
      rules = {}
      if promotion_type == 'sku':
          rules = {'sku': f"SKU{fake.random_int(1, 50000):06d}"}
      elif promotion_type == 'category':
          rules = {'category_id': fake.random_int(1, 50)}
      elif promotion_type == 'perishable':
          rules = {'expires_days': fake.random_int(1, 7)}
      Promotion.objects.create(
          type=promotion_type,
          percentage=fake.random_int(10, 50) / 100,
          valid_until=fake.date_time_between(start_date='now', end_date='+30d'),
          rules=rules,
          min_order=fake.random_int(10, 100) if promotion_type == 'coupon' else 0
      )
  ```
- **Seed Elasticsearch**:
  ```javascript
  // scripts/seed_elasticsearch.js
  const { Client } = require('@elastic/elasticsearch');
  const client = new Client({ node: 'http://localhost:9200' });

  async function seed() {
    const products = await fetch('http://product-catalog:8000/products').then(res => res.json());
    for (const product of products) {
      await client.index({
        index: 'products',
        id: product.sku,
        body: {
          sku: product.sku,
          name_en: product.name_en,
          name_vi: product.name_vi,
          category: product.category_id,
          price: product.price,
          dietary: product.dietary,
          tags: product.tags,
          is_perishable: product.is_perishable,
          expires_at: product.expires_at,
        },
      });
    }
    console.log('Elasticsearch seeded');
  }

  seed().catch(console.error);
  ```

**Resource Considerations**:
- **Laptop Specs**: Allocate ~10GB RAM for containers (512MB each for Product Catalog, Inventory, Order, Delivery, Recommendation; 256MB for Payment, User, Notification; 1GB for Elasticsearch; 512MB for Kafka; 2GB for dev container/`kind`).
- **Optimization**: Stop unused containers (`docker stop recommendation`) during development. Use `docker stats` to monitor CPU/RAM. Run Elasticsearch with minimal settings (`-Xms512m -Xmx1g`).
- **Constraints**: Ensure Docker supports ~12 containers. Use WSL2 (Windows) for performance if needed.

**Output**: Comprehensive microservices with scalable REST APIs, Kafka-driven events, Redis caching, and grocery-specific features (perishable tracking, substitutions, same-day slots, promotions, advanced search, recommendations). Docker images deployed to `kind` via Helm, integrated with CI/CD, tested for ≥85% coverage, ready for Phase 3.

### Phase 3: Frontend and UX (3–4 Weeks)
**Goal**: Develop a mobile-first frontend with grocery-specific UX (search, checkout, delivery slots, promotions, recommendations).

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 3.1 Prototype | Design UX in Figma for search, cart, checkout, slots, promotions, recommendations. | 1 week | None | Mobile-first, multilingual (English/Vietnamese). Single-page checkout, calendar slot picker, promotion banners, recommendation widgets. |
| 3.2 Build frontend | Develop React frontend, connect to GraphQL/REST APIs. | 2 weeks | 2.1–2.9, 3.1 | Use Apollo Client for GraphQL. Components: `ProductList`, `Cart`, `SlotPicker`, `PromotionBanner`, `RecommendationWidget`. Cache with Apollo. |
| 3.3 Multilingual | Implement English/Vietnamese toggle. | 3 days | 3.2 | Use `react-i18next`. Fetch translations from Product Catalog API. |
| 3.4 Optimize | Ensure ≤2s page loads with Lighthouse. | 3 days | 3.2 | Compress images, use Cloudflare CDN, lazy-load products/recommendations. |

**Grocery Focus**: Streamlined checkout with substitution/promotion options, dietary/perishable filters, popular items, recommendations, fast slot selection (≤500ms).
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
query GetRecommendations($userId: String!) {
  recommendations(userId: $userId) {
    skus
  }
}
```
**Output**: Responsive frontend integrated with backend, supporting all features.

### Phase 4: Third-Party Integrations (2–3 Weeks)
**Goal**: Finalize Stripe, MoMo, Grab, ViettelPost, and Gojek integrations.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 4.1 Stripe/MoMo | Complete Stripe.js/MoMo SDK integration, test webhooks. | 1 week | 2.4 | NestJS, `@stripe/stripe-js`, MoMo SDK. Kafka topic: `payment_events`. Test with sandboxes. |
| 4.2 Grab/ViettelPost/Gojek | Integrate delivery APIs, test slot booking. | 1.5 weeks | 2.6 | NestJS, REST APIs. Cache slots in Redis. Test with provider sandboxes. |
| 4.3 Extensibility | Abstract Payment/Delivery Services for future providers. | 3 days | 4.1, 4.2 | Strategy pattern in NestJS: `PaymentProcessor`, `DeliveryProvider`. Store configs in Vault. |

**Grocery Focus**: Ensure ≤1s payment processing, ≤500ms slot updates, reliable webhook handling.
**Output**: Fully integrated third-party services, extensible for future providers.

### Phase 5: Testing and Validation (3–4 Weeks)
**Goal**: Ensure reliability, performance, and grocery-specific functionality through comprehensive testing.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 5.1 Unit tests | Write tests for all services (≥80% coverage). | 1 week | 2.1–2.9 | pytest for Django, Jest for NestJS. Test promotions, search, recommendations. |
| 5.2 Integration tests | Test service interactions (REST, Kafka). | 1 week | 2.1–2.9, 4.1–4.3 | Testcontainers for PostgreSQL/Firestore/Elasticsearch. Mock Stripe/MoMo/Grab APIs with `nock`. |
| 5.3 E2E tests | Test user flows (search, checkout, slots, recommendations). | 1 week | 3.2, 5.2 | Cypress for browser tests. Test multilingual UX, substitutions, promotions. |
| 5.4 Load tests | Simulate 300,000 users, validate performance. | 3 days | 5.2 | Locust: Test ≤500ms search, ≤1s checkout. Monitor with Prometheus/Grafana. |
| 5.5 Security tests | Scan for vulnerabilities, test JWT/RBAC. | 2 days | 5.2 | OWASP ZAP for APIs, Snyk for dependencies. |

**Grocery Focus**: Test inventory accuracy (≥99%), slot booking success (≥98%), promotion/recommendation accuracy under load.
**Output**: Fully tested platform meeting performance and reliability goals.

### Phase 6: Deployment and Beta Launch (2–3 Weeks)
**Goal**: Deploy the platform, conduct beta testing, and prepare for production.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 6.1 Staging deploy | Deploy to staging Kubernetes cluster. | 3 days | 2.1–2.9, 3.2, 4.1–4.3 | Use Helm: `helm upgrade grocery ./charts`. Blue-green for Inventory/Delivery. |
| 6.2 Beta testing | Launch to 100–500 users, collect feedback. | 1 week | 6.1, 5.1–5.5 | Use Django `waffle` for feature flags. Google Forms, Hotjar for feedback. |
| 6.3 Fix issues | Address beta feedback (e.g., UX, performance). | 1 week | 6.2 | Prioritize critical issues (e.g., slow checkout >1s). Update GitHub Issues. |
| 6.4 Production deploy | Deploy to production, monitor KPIs. | 3 days | 6.3 | Blue-green deployment, smoke tests (Postman). Monitor with Prometheus/Grafana. |

**Grocery Focus**: Validate real-time inventory (≤100ms), delivery slots (≤500ms), promotion/recommendation UX during beta.
**Output**: Production-ready platform, initial KPIs (e.g., conversion rate ≥3%).

## Timeline
- **Total Duration**: 25–34 weeks (6–8 months) for all phases.
- **Breakdown**:
  - Phase 1: 1–2 weeks
  - Phase 2: 14–16 weeks
  - Phase 3: 3–4 weeks
  - Phase 4: 2–3 weeks
  - Phase 5: 3–4 weeks
  - Phase 6: 2–3 weeks
- **Assumption**: 20–30 hours/week, with flexibility for adjustments.

## Best Practices for Solo Development
- **Prioritization**: Focus on core services (Product Catalog, Inventory, Order, Payment, User, Delivery, Notification) first, then Search, Recommendation, and additional integrations (MoMo/Gojek).
- **Automation**:
  - Use GitHub Actions for CI/CD (lint, test, deploy).
  - Deploy monitoring (Prometheus/Grafana, Loki) early.
  - Script tasks (e.g., `seed_promotions.py`, `seed_elasticsearch.js`) in `/scripts`.
- **Task Management**: Use GitHub Projects with Kanban board. Label tasks (e.g., `priority:high`, `grocery`) and set milestones (“Phase 2”).
- **Technical Debt**:
  - Allocate 10% of time to refactoring (e.g., optimize Elasticsearch queries).
  - Use Snyk to catch vulnerabilities.
- **Grocery-Specific**:
  - Test inventory locking, substitutions, promotions rigorously.
  - Cache delivery slots (Redis) for ≤500ms responses.
  - Validate multilingual UX, search, recommendations in beta.
- **Cloud Prep**: Use cloud-agnostic tools (MinIO, Helm) and test staging for AWS/GCP.

## Monitoring and Validation
- **KPIs** (Phase 6):
  - Conversion Rate: ≥3%.
  - Order Fulfillment: ≥95%.
  - Inventory Accuracy: ≥99%.
  - Slot Booking Success: ≥98%.
  - Uptime: ≥99.9%.
  - Recommendation Click-Through: ≥5%.
- **Tools**: Prometheus/Grafana for metrics, GA4 for behavior, Hotjar for UX, Firestore for events.
- **Feedback**: Collect beta feedback via Google Forms/Hotjar to refine UX.

## Risks and Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| Overrunning timeline | Delays launch | Automate tests, use GitHub Projects, prioritize core services. |
| Performance issues | Slow UX (>1s checkout) | Cache with Redis, optimize Elasticsearch, test with Locust. |
| Third-party failures | Payment/delivery disruptions | Use circuit breakers, mock APIs, cache responses. |
| Technical debt | Maintenance burden | Refactor incrementally, enforce ≥80% coverage, use Snyk. |
| Resource constraints | Slow development | Stop unused containers, optimize Elasticsearch RAM, use WSL2. |

## Next Steps
- Start Phase 1 by initializing repo and Kubernetes cluster.
- Review tasks in GitHub Projects weekly, adjusting priorities.
- Deploy monitoring early (Phase 1) to track progress.
- Test all features (promotions, search, recommendations) in Phase 2 to ensure grocery-specific functionality.
