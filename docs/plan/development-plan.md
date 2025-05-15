# Development Plan

## Overview
This plan outlines the development of a large-scale, microservices-based grocery e-commerce platform supporting 100,000 daily active users (DAU), 300,000 peak users, and 50,000 SKUs. The platform prioritizes scalability, performance (≤500ms search, ≤2s page load, ≤100ms inventory updates, 99.9% uptime), and grocery-specific features (e.g., real-time inventory, same-day delivery, substitutions, multilingual English/Vietnamese UX, promotions, advanced search, personalized recommendations). It now includes additional e-commerce features like customer reviews, wishlists, returns management, loyalty programs, analytics, social media integration, order history, guest checkout, product bundling/subscriptions, and live chat. The platform runs on Kubernetes/Docker on-premises, with future cloud migration (e.g., AWS EKS). As a solo developer, the plan focuses on automation, prioritization, and modularity to streamline implementation over 29–38 weeks (7–9 months) at 20–30 hours/week.

## Scope
- **Features**:
  - Product Catalog: Browse/search 50,000 SKUs, multilingual descriptions, dietary/expiry filters, popular items, promotions, customer reviews, product bundling.
  - Inventory: Real-time stock updates (≤100ms), perishable tracking, substitutions.
  - Order: Cart, checkout (≤1s), order tracking, discounts, coupons, returns, order history, guest checkout, subscriptions.
  - Payment: Stripe and MoMo integration, PCI DSS compliant, subscription billing.
  - User: JWT-based authentication, profiles, wishlists, loyalty programs, social media login.
  - Delivery: Grab, ViettelPost, Gojek integration, same-day slots (≤500ms), in-store pickup.
  - Notifications: Order/delivery updates (email/SMS).
  - Search: Elasticsearch-based advanced search with fuzzy matching, dietary filters.
  - Recommendation: Personalized suggestions with rule-based and collaborative filtering.
  - Analytics: Dashboards for sales, inventory, user behavior.
  - Live Chat: Real-time customer support via Zendesk or similar.
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
- **Third-Party**: Stripe/MoMo (payments), Grab/ViettelPost/Gojek (delivery), SendGrid/Twilio (notifications), Zendesk (live chat), Metabase (analytics), Firebase Authentication (social login).

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

### Phase 2: Core Microservices and Feature Development (18–20 Weeks)
**Goal**: Build a comprehensive set of microservices (Product Catalog, Inventory, Order, Payment, User, Delivery, Notification, Search, Recommendation, Analytics, Live Chat) with all features (promotions, advanced search, personalized recommendations, MoMo/Gojek integrations, customer reviews, wishlists, returns, loyalty programs, social media integration, order history, guest checkout, product bundling/subscriptions, analytics, live chat) for a large-scale grocery e-commerce platform. Ensure scalability, performance, and grocery-specific functionality (e.g., perishable tracking, substitutions, same-day delivery, dietary search, promotions). Each service includes APIs, database models, Kafka event handling, caching, rate limiting, circuit breakers, health checks, and automated tests, leveraging Phase 1’s Docker environment. Optimized for a solo developer to support 100,000 DAU and 300,000 peak users.

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
| 2.1 Product Catalog | Build a scalable service for 50,000 SKUs with multilingual descriptions, dietary/expiry filters, popular items, promotions, customer reviews, and product bundling. | 3 weeks | 1.4 | **Tech**: Django 4.2, PostgreSQL, `django-environ`, `django-filter`, `django-rest-framework`. **Models**: `Product(sku: str, name_en: str, name_vi: str, category_id: fk, price: decimal, is_perishable: bool, expires_at: datetime, stock_updated_at: datetime, dietary: array[str], tags: array[str], unit_of_measure: str, brand: str, manufacturer: str, image_urls: array[url], average_rating: decimal)`, `Category(id: uuid, name_en: str, name_vi: str)`, `Promotion(id: uuid, type: str, percentage: decimal, valid_until: datetime, rules: jsonb, min_order: decimal)`, `Review(id: uuid, product_id: fk, user_id: fk, rating: int, comment: text, created_at: datetime)`, `Bundle(id: uuid, name_en: str, name_vi: str, description: text, included_products: array[sku], price: decimal)`. **API**: REST (`/products`, `/products/{sku}`, `/categories`, `/products/popular`, `/promotions`, `/products/promoted`, `/products/{sku}/images`, `/products/{sku}/reviews`, `/bundles`, `/bundles/{id}`, GET/POST/PUT/DELETE) with pagination, filtering (`category`, `is_perishable`, `dietary`, `expires_soon`, `tags`, `price_range`, `min_rating`). **Search**: PostgreSQL GIN index on `name_en`, `name_vi`, `dietary`, `tags` for ≤500ms queries; sync with Elasticsearch (2.8). **Elasticsearch Mapping**: Fields: `sku`, `name_en`, `name_vi`, `category`, `price`, `dietary`, `tags`, `is_perishable`, `expires_at`, `average_rating`; custom `vi_analyzer` for Vietnamese. **Popular Items**: `SELECT sku, COUNT(*) FROM order_items WHERE created_at > now() - interval '7 days' GROUP BY sku ORDER BY COUNT DESC LIMIT 10`. **Promotions**: Support SKU-based, category-based, perishable-specific, buy-one-get-one, bulk discounts (`rules: {"sku": "milk", "expires_days": 3, "min_quantity": 2}`). **Customer Reviews**: Allow authenticated users to submit 1–5 star ratings and comments; calculate `average_rating` on save; prevent duplicate reviews per user. **Product Bundling**: Create bundles as special products; validate stock for included SKUs during checkout; support bundle-specific promotions. **Caching**: Redis (`products:{sku}`, `popular:{category}`, `promotions:{id}`, `reviews:{sku}`, `bundles:{id}`, 1-hour TTL). **Rate Limiting**: NGINX (`limit_req zone=products burst=50`). **Circuit Breaker**: `tenacity` for Elasticsearch (retry 3x, timeout 500ms). **Health Check**: `/health`. **Kafka**: Publish `ProductUpdated`, `PromotionUpdated`, `ReviewAdded` to `product_events`. **Scripts**: Seed 50,000 SKUs (`seed_products.py`, 20% perishables, 10% vegan/gluten-free), promotions (`seed_promotions.py`), bundles (`seed_bundles.py`), mock reviews (`seed_reviews.py`). **Tests**: pytest for filters, popular items, promotions, reviews, bundles (≥85% coverage); Testcontainers for PostgreSQL. **Metrics**: Prometheus for filter usage, cache hits, review submissions, bundle purchases. **Logging**: Log slow queries (>300ms) to Sentry. **Grocery**: Support dietary filters, near-expiry highlights, popular items, perishable promotions, review-driven trust, bundled meal kits. **Scalability**: Shard PostgreSQL by `category_id`; index `dietary`, `tags`, `average_rating`. **HA**: 2 replicas in `kind` via Helm (`charts/product-catalog`). **Debug**: Use Django Debug Toolbar locally. **CI/CD**: GitHub Actions (`lint`, `test`, `build`, `push`). **Docker**: Image (`grocery/product-catalog:1.0`, ~200MB). **Resource**: 512MB RAM. |
| 2.2 Inventory | Build a real-time stock management service with perishable tracking, optimistic locking, and Kafka integration. | 2 weeks | 1.4, 1.5 | **Tech**: NestJS 10, Firestore emulator, Kafka (`bitnami/kafka:3.7`), `nestjs-throttler`. **Model**: Firestore `inventory/{location}/{sku}` with `{stock: number, reserved_stock: number, expires_at: timestamp, last_updated: timestamp, low_stock_threshold: number, location: str}`. **API**: REST (`/stock/{sku}`, `/stock/batch`, `/stock/{sku}/reserve`, `/stock/{sku}/release`, GET/POST, ≤100ms). **Kafka**: Deploy single-broker Kafka in Docker (`docker run -d --memory=512m bitnami/kafka`), topics: `order_events`, `product_events`. Consume `OrderPlaced`, `ReturnProcessed` to adjust stock, `ProductUpdated` to sync SKUs. Publish `StockUpdated` on low stock. **Logic**: Optimistic locking via Firestore transactions; alert on low stock (<`low_stock_threshold`); restock returned items. **Caching**: Redis (`stock:{sku}`, 30s TTL). **Rate Limiting**: `nestjs-throttler` (100 req/s). **Circuit Breaker**: `opossum` for Firestore calls (timeout 200ms). **Health Check**: `/health` (Firestore, Kafka, Redis). **Scripts**: Seed 50,000 SKUs (`seed_inventory.js`), simulate stock updates (`test_stock.js`). **Tests**: Jest for APIs, transactions, Kafka consumers (≥85% coverage); mock Kafka with `kafkajs`, Firestore with emulator. **Metrics**: Prometheus (`prom-client`) for stock update latency, error rates. **Logging**: Winston to Loki (deferred to Phase 5), console locally. **Grocery**: Track expiration, support multi-location stock, restock perishables from returns. **Scalability**: Partition Firestore by `location`. **HA**: 2 replicas in `kind` (`charts/inventory`). **Debug**: Trace Firestore transactions with OpenTelemetry (deferred to Phase 5). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/inventory:1.0`, ~150MB). **Resource**: 512MB RAM (Kafka 512MB). |
| 2.3 Order | Build cart, checkout, and order tracking with substitutions, discounts, coupons, returns management, order history, guest checkout, and subscriptions. | 4 weeks | 2.1, 2.2 | **Tech**: Django, PostgreSQL, `django-redis`, `requests`. **Models**: `Order(id: uuid, user_id: str, guest_email: str, guest_address: jsonb, items: jsonb, status: str, substitution_prefs: jsonb, promotion_id: fk, discount_applied: decimal, coupon_code: str, created_at: datetime, delivery_instructions: text, return_requested: bool, return_status: str, return_reason: text)`, `Cart(user_id: str, guest_id: str, items: jsonb, updated_at: datetime)`, `Subscription(id: uuid, user_id: str, items: jsonb, frequency: str, next_delivery: datetime, status: str)`. **API**: REST (`/cart`, `/cart/{user_id}`, `/checkout`, `/guest-checkout`, `/orders/{id}`, `/orders/history`, `/orders/{id}/apply-coupon`, `/orders/{id}/confirm-substitutions`, `/orders/{id}/return`, `/orders/{id}/return/status`, `/orders/reorder/{id}`, `/subscriptions`, `/subscriptions/{id}`, GET/POST, ≤1s). **Kafka**: Publish `OrderPlaced`, `ReturnRequested`, `SubscriptionCreated` to `order_events`. Consume `PaymentProcessed`, `ReturnProcessed` to update status. **Logic**: Validate stock via Inventory API, apply substitutions (e.g., `{sku: "milk", allow: true, alt_sku: "milk2"}`), fetch promotions from Product Catalog (`/promotions`), apply highest discount if valid (`min_order` met, `valid_until` > now), update `status` (`pending`, `confirmed`, `fulfilled`). **Returns Management**: Allow users to request returns within 7 days; update `return_status` (`pending`, `approved`, `rejected`, `completed`); trigger refund via Payment service; restock via Inventory. **Order History**: Retrieve past orders for authenticated/guest users (via `user_id` or `guest_email`). **Reordering**: Copy past order items to cart. **Guest Checkout**: Collect guest details (email, address); generate temporary `guest_id` for cart; store in Order model. **Subscriptions**: Create recurring orders with weekly/monthly frequency; schedule `next_delivery`; integrate with Payment for recurring billing. **Caching**: Redis (`cart:{user_id}`, `cart:guest:{guest_id}`, `promotions:{id}`, `coupons:{code}`, `orders:{user_id}`, 24-hour/1-hour TTL). **Rate Limiting**: NGINX (`limit_req zone=order burst=20`). **Circuit Breaker**: `tenacity` for Inventory/Product Catalog APIs (retry 3x, timeout 500ms). **Health Check**: `/health` (DB, Redis, Inventory, Product Catalog). **Scripts**: Mock checkout (`test_checkout.py`), simulate substitutions (`test_substitutions.py`), seed promotions/coupons (`seed_promotions.py`), test returns (`test_returns.py`), test subscriptions (`test_subscriptions.py`). **Tests**: pytest for cart, checkout, substitutions, promotions, coupons, returns, history, guest checkout, subscriptions (≥85% coverage); Testcontainers for PostgreSQL. **Metrics**: Prometheus for checkout latency, promotion usage, coupon application, return rates, subscription activations. **Logging**: Log checkout/promotion/coupon/return errors to Sentry. **Grocery**: Support per-item substitutions, real-time stock validation, auto-apply perishable discounts/coupons, handle returns for perishables, enable recurring grocery subscriptions. **Scalability**: Index `user_id`, `guest_email`, `promotion_id`, `coupon_code`. **HA**: 2 replicas in `kind` (`charts/order`). **Debug**: Log API call latencies (>800ms). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/order:1.0`, ~200MB). **Resource**: 512MB RAM. |
| 2.4 Payment | Integrate Stripe and MoMo for secure payments, including refunds and subscription billing. | 2.5 weeks | 2.3 | **Tech**: NestJS, Stripe SDK (`@stripe/stripe-js`), MoMo SDK, PostgreSQL. **Model**: `Payment(id: uuid, order_id: uuid, subscription_id: uuid, provider: str, provider_id: str, amount: decimal, status: str, created_at: datetime)`. **API**: REST (`/pay`, `/payment/{order_id}`, `/payments/{id}/refund`, `/subscriptions/{id}/billing`, GET/POST, ≤1s). **Kafka**: Publish `PaymentProcessed`, `SubscriptionBilled` to `payment_events`. **Logic**: Implement `PaymentProcessor` interface (`processPayment`, `handleWebhook`, `processSubscriptionBilling`) for Stripe and MoMo; create PaymentIntent for Stripe, equivalent for MoMo; handle webhooks (`payment_intent.succeeded`, `payment_intent.failed`, `invoice.payment_succeeded` for Stripe; MoMo equivalents) with idempotency; process recurring subscription payments via Stripe Subscriptions or MoMo recurring payments. **Security**: PCI DSS via Stripe.js/MoMo SDK, no card data stored. **Rate Limiting**: `nestjs-throttler` (50 req/s). **Circuit Breaker**: `opossum` for provider APIs (timeout 500ms). **Health Check**: `/health` (DB, Stripe, MoMo). **Scripts**: Test webhooks (`test_payment.js`), simulate payments (`simulate_payment.js`), test subscription billing (`test_subscription_billing.js`). **Tests**: Jest for APIs, webhooks, subscription billing, provider switching (≥85% coverage); mock Stripe with `stripe-mock`, MoMo with `nock`. **Metrics**: Prometheus for payment success rate, latency, provider usage, subscription billing success. **Logging**: Log webhook errors to Sentry. **Grocery**: Ensure fast payment processing, support subscription billing for recurring orders. **Scalability**: Async webhooks and billing tasks. **HA**: 2 replicas in `kind` (`charts/payment`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/payment:1.0`, ~150MB). **Resource**: 256MB RAM. |
| 2.5 User | Build JWT-based authentication, user profiles, wishlists, loyalty programs, and social media login. | 2 weeks | 1.4 | **Tech**: Django, `djangorestframework-simplejwt`, `django-allauth` (social login), PostgreSQL. **Model**: `User(id: uuid, email: str, password: str, role: str, name_en: str, name_vi: str, dietary_preferences: array[str], favorite_products: array[str], addresses: jsonb, loyalty_points: decimal, social_provider: str, social_id: str)`, `LoyaltyTransaction(id: uuid, user_id: fk, points: decimal, type: str, order_id: fk, created_at: datetime)`. **API**: REST (`/login`, `/refresh`, `/profile`, `/users/{id}`, `/wishlist`, `/wishlist/{sku}`, `/loyalty/points`, `/loyalty/transactions`, `/social/login`, POST/GET/DELETE). **Logic**: Issue JWT tokens (access: 15m, refresh: 24h), RBAC with roles (`customer`, `admin`, `staff`); support Google/Facebook login via `django-allauth`; manage wishlist (`favorite_products`); award loyalty points (e.g., 1 point per $1 spent, redeemable as discounts); track loyalty transactions. **Security**: Hash passwords with `argon2`, enforce TLS 1.3, secure OAuth2 flows. **Rate Limiting**: NGINX (`limit_req zone=user burst=100`). **Health Check**: `/health`. **Kafka**: Publish `UserUpdated`, `LoyaltyPointsAwarded` to `user_events`. **Scripts**: Seed test users (`seed_users.py`), test RBAC (`test_rbac.py`), test loyalty (`test_loyalty.py`). **Tests**: pytest for auth, RBAC, wishlist, loyalty, social login (≥85% coverage). **Metrics**: Prometheus for login success rate, token issuance, loyalty point redemptions. **Logging**: Log auth failures, social login errors to Sentry. **Grocery**: Support multilingual names, role-based order access, dietary preferences, loyalty incentives for frequent purchases. **Scalability**: Index `email`, `social_id`. **HA**: 2 replicas in `kind` (`charts/user`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/user:1.0`, ~200MB). **Resource**: 256MB RAM. |
| 2.6 Delivery | Integrate Grab, ViettelPost, and Gojek for same-day delivery slots, in-store pickup, and real-time tracking. | 2.5 weeks | 2.3 | **Tech**: NestJS, Firestore, Redis, `nestjs-throttler`. **Model**: Firestore `delivery/{orderId}` with `{slot: timestamp, provider: str, status: str, tracking_url: str, pickup: bool}`. **API**: REST (`/slots`, `/delivery/{orderId}`, `/pickup`, GET/POST, ≤500ms). **Caching**: Redis (`slots:{date}:{provider}`, 1-hour TTL). **Kafka**: Publish `DeliveryScheduled`, `DeliveryUpdated` to `delivery_events`. **Logic**: Implement `DeliveryProvider` interface (`fetchSlots`, `bookDelivery`, `trackDelivery`) for Grab, ViettelPost, Gojek; mock APIs initially, fallback to next-day slots, support pickup locations; handle return logistics for approved returns. **Rate Limiting**: `nestjs-throttler` (200 req/s). **Circuit Breaker**: `opossum` for provider APIs (timeout 300ms). **Health Check**: `/health`. **Scripts**: Seed slots (`seed_slots.js`), test pickup (`test_pickup.js`), test provider switching (`test_delivery_providers.js`). **Tests**: Jest for slot booking, cache hits, provider switching (≥85% coverage). **Metrics**: Prometheus for slot booking success, latency, provider usage. **Logging**: Log cache misses, API errors to Sentry. **Grocery**: Prioritize same-day slots, display tracking URLs, support return pickups. **Scalability**: Partition Firestore by `orderId`. **HA**: 2 replicas in `kind` (`charts/delivery`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/delivery:1.0`, ~150MB). **Resource**: 512MB RAM. |
| 2.7 Notification | Build scalable email/SMS notifications for order, delivery, and loyalty updates. | 1 week | 2.3, 2.5, 2.6 | **Tech**: NestJS, Firestore, SendGrid/Twilio, `nestjs-throttler`. **Model**: Firestore `notifications/{id}` with `{order_id: str, user_id: str, type: str, status: str, sent_at: timestamp, payload: object}`. **API**: Internal (`/notify`, POST). **Kafka**: Consume `OrderConfirmed`, `DeliveryScheduled`, `PaymentProcessed`, `ReturnProcessed`, `LoyaltyPointsAwarded`. **Logic**: Send multilingual email/SMS with order status, substitutions, promotions, delivery details, loyalty points earned/redeemed. **Rate Limiting**: `nestjs-throttler` (50 req/s). **Circuit Breaker**: `opossum` for SendGrid/Twilio APIs (timeout 500ms). **Health Check**: `/health`. **Scripts**: Test notifications (`test_notify.js`), simulate failures (`test_notify_fail.js`). **Tests**: Jest for message delivery, payload parsing (≥85% coverage); mock SendGrid/Twilio. **Metrics**: Prometheus for notification success rate. **Logging**: Log delivery failures to Sentry. **Grocery**: Include order status, substitutions, promotions, loyalty updates in messages. **Scalability**: Batch notifications in Kafka consumer. **HA**: 2 replicas in `kind` (`charts/notification`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/notification:1.0`, ~150MB). **Resource**: 256MB RAM. |
| 2.8 Search | Implement advanced search with Elasticsearch for 50,000 SKUs, including review-based filtering. | 2 weeks | 2.1 | **Tech**: NestJS, Elasticsearch (`docker.elastic.co/elasticsearch:8.10.0`), `nestjs-throttler`. **Setup**: Deploy Elasticsearch in Docker (`docker run -d --memory=1g elasticsearch:8.10.0`). **Index**: `products` index with fields (`sku`, `name_en`, `name_vi`, `category`, `price`, `dietary`, `tags`, `is_perishable`, `expires_at`, `average_rating`). **API**: REST (`/search`, GET, ≤500ms) with fuzzy search, filters (`dietary`, `price_range`, `expires_soon`, `min_rating`). **Logic**: Sync PostgreSQL products to Elasticsearch via Kafka (`ProductUpdated`, `ReviewAdded`); handle partial matches (e.g., “mil” → “milk”). **Caching**: Redis (`search:{query}`, 1-hour TTL). **Rate Limiting**: `nestjs-throttler` (100 req/s). **Health Check**: `/health`. **Scripts**: Seed index (`seed_elasticsearch.js`), test fuzzy search (`test_search.js`). **Tests**: Jest for search queries, sync logic, review-based filtering (≥85% coverage). **Metrics**: Prometheus for search latency, hit rate. **Logging**: Log search errors to Sentry. **Grocery**: Support dietary/perishable filters, fuzzy search for Vietnamese terms, rating-based product ranking. **Scalability**: Single-node Elasticsearch; scale to cluster post-MVP. **HA**: 1 replica in `kind` (`charts/search`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/search:1.0`, ~150MB). **Resource**: 1GB RAM (Elasticsearch). |
| 2.9 Recommendation | Build personalized suggestions with rule-based, collaborative filtering, and review-based logic. | 2 weeks | 2.1, 2.3 | **Tech**: Django, PostgreSQL, `django-environ`. **Model**: `Recommendation(id: uuid, user_id: str, skus: array[str], created_at: datetime)`. **API**: REST (`/recommendations/{user_id}`, GET, ≤500ms). **Logic**: Rule-based (suggest SKUs from user’s category preferences, `dietary_preferences`, `favorite_products`); collaborative filtering (item-item similarity: `SELECT sku2, COUNT(*) FROM order_items oi1 JOIN order_items oi2 ON oi1.order_id = oi2.order_id WHERE oi1.sku = :sku GROUP BY sku2`); review-based (prioritize high-rated products: `SELECT sku FROM reviews WHERE rating >= 4 GROUP BY sku ORDER BY AVG(rating) DESC`). **Caching**: Redis (`recommendations:{user_id}`, 1-hour TTL). **Rate Limiting**: NGINX (`limit_req zone=recommendations burst=50`). **Health Check**: `/health`. **Scripts**: Seed recommendations (`seed_recommendations.py`), test suggestions (`test_recommendations.py`). **Tests**: pytest for rules, collaborative filtering, review-based logic (≥85% coverage). **Metrics**: Prometheus for recommendation latency, click-through rate. **Logging**: Log errors to Sentry. **Grocery**: Suggest perishables, dietary-compatible items, high-rated products. **Scalability**: Index `user_id`, `sku`. **HA**: 2 replicas in `kind` (`charts/recommendation`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/recommendation:1.0`, ~200MB). **Resource**: 512MB RAM. |
| 2.10 Analytics | Build admin dashboards for business metrics (sales, inventory, user behavior). | 1.5 weeks | 2.1, 2.3, 2.5 | **Tech**: NestJS, PostgreSQL, Metabase (`metabase/metabase:latest`), `nestjs-throttler`. **Setup**: Deploy Metabase in Docker (`docker run -d --memory=512m metabase/metabase`). **API**: REST (`/analytics/sales`, `/analytics/inventory`, `/analytics/users`, `/analytics/promotions`, GET, ≤500ms) for admin access. **Logic**: Aggregate data from PostgreSQL (`orders`, `products`, `users`, `promotions`); calculate metrics (e.g., total sales, top SKUs, user retention, promotion ROI); expose to Metabase for visualization. **Queries**: Sales: `SELECT SUM(discount_applied) as revenue FROM orders WHERE created_at > now() - interval '30 days'`; Inventory: `SELECT sku, stock FROM inventory WHERE stock < low_stock_threshold`; Users: `SELECT COUNT(DISTINCT user_id) FROM orders WHERE created_at > now() - interval '7 days'`. **Caching**: Redis (`analytics:{metric}`, 1-hour TTL). **Rate Limiting**: `nestjs-throttler` (50 req/s). **Health Check**: `/health`. **Scripts**: Seed analytics data (`seed_analytics.js`), test queries (`test_analytics.js`). **Tests**: Jest for API responses, query accuracy (≥85% coverage). **Metrics**: Prometheus for query latency, cache hits. **Logging**: Log query errors to Sentry. **Grocery**: Track perishable turnover, promotion effectiveness, subscription uptake. **Scalability**: Index `created_at`, `sku`. **HA**: 1 replica in `kind` (`charts/analytics`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/analytics:1.0`, ~150MB). **Resource**: 512MB RAM (Metabase 512MB). |
| 2.11 Live Chat | Integrate Zendesk for real-time customer support. | 1 week | 2.5 | **Tech**: NestJS, Zendesk SDK, Firestore, `nestjs-throttler`. **Model**: Firestore `support_tickets/{id}` with `{user_id: str, order_id: str, issue: str, status: str, created_at: timestamp, messages: array[object]}`. **API**: REST (`/support/tickets`, `/support/tickets/{id}`, `/support/messages`, POST/GET, ≤500ms). **Logic**: Integrate Zendesk Chat widget; create tickets for user inquiries; sync messages to Firestore; notify admins via Notification service. **Rate Limiting**: `nestjs-throttler` (50 req/s). **Circuit Breaker**: `opossum` for Zendesk API (timeout 500ms). **Health Check**: `/health`. **Scripts**: Test ticket creation (`test_support.js`), simulate chats (`test_chat.js`). **Tests**: Jest for ticket creation, message syncing (≥85% coverage); mock Zendesk with `nock`. **Metrics**: Prometheus for ticket creation rate, resolution time. **Logging**: Log API errors to Sentry. **Grocery**: Support inquiries about orders, returns, subscriptions. **Scalability**: Partition Firestore by `user_id`. **HA**: 1 replica in `kind` (`charts/live-chat`). **CI/CD**: GitHub Actions. **Docker**: Image (`grocery/live-chat:1.0`, ~150MB). **Resource**: 256MB RAM. |

**Grocery Focus**:
- **Inventory**: Prevent oversells with Firestore transactions, track perishable expiration, alert on low stock, restock returns.
- **Order**: Support substitutions, validate stock, apply discounts/coupons, manage returns, provide order history, enable guest checkout, offer subscriptions.
- **Delivery**: Optimize same-day slots, support multiple providers, handle return logistics.
- **Product Catalog**: Offer dietary/expires_soon filters, popular items, promotions, reviews, bundled meal kits.
- **Search**: Enable fuzzy search, dietary/perishable/rating filters.
- **Recommendation**: Suggest perishables, dietary-compatible, high-rated items.
- **Payment**: Support Stripe/MoMo, process subscription billing.
- **User**: Manage wishlists, loyalty points, social logins.
- **Notification**: Deliver multilingual messages with order, substitution, promotion, loyalty, return details.
- **Analytics**: Track sales, inventory turnover, user retention, promotion effectiveness.
- **Live Chat**: Handle customer inquiries about orders, returns, subscriptions.
- **Performance**: Ensure ≤100ms inventory updates, ≤500ms search/slots, ≤1s checkout.
- **Reliability**: Achieve ≥99% inventory accuracy, ≥98% slot booking success, ≥95% promotion accuracy.

**Code Snippets**:
- **Product Catalog (Reviews and Bundles)**:
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
      average_rating = models.DecimalField(max_digits=3, decimal_places=2, default=0)

  class Review(models.Model):
      id = models.UUIDField(primary_key=True, default=uuid.uuid4)
      product = models.ForeignKey('Product', on_delete=models.CASCADE)
      user = models.ForeignKey('user.User', on_delete=models.CASCADE)
      rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
      comment = models.TextField(blank=True)
      created_at = models.DateTimeField(auto_now_add=True)

      class Meta:
          unique_together = ['product', 'user']

  class Bundle(models.Model):
      id = models.UUIDField(primary_key=True, default=uuid.uuid4)
      name_en = models.CharField(max_length=255)
      name_vi = models.CharField(max_length=255)
      description = models.TextField()
      included_products = ArrayField(models.CharField(max_length=50))
      price = models.DecimalField(max_digits=10, decimal_places=2)
  ```
  ```python
  # services/product-catalog/product/views.py
  from rest_framework import viewsets
  from django.contrib.postgres.search import SearchVector
  from django_filters.rest_framework import DjangoFilterBackend
  from django.utils import timezone
  from datetime import timedelta
  from .models import Product, Review, Bundle
  from .serializers import ProductSerializer, ReviewSerializer, BundleSerializer

  class ReviewViewSet(viewsets.ModelViewSet):
      queryset = Review.objects.all()
      serializer_class = ReviewSerializer

      def perform_create(self, serializer):
          serializer.save(user=self.request.user)
          product = serializer.validated_data['product']
          reviews = Review.objects.filter(product=product)
          product.average_rating = sum(r.rating for r in reviews) / reviews.count()
          product.save()

  class BundleViewSet(viewsets.ModelViewSet):
      queryset = Bundle.objects.all()
      serializer_class = BundleSerializer
  ```
- **Order (Returns, Subscriptions, Guest Checkout)**:
  ```python
  # services/order/order/models.py
  from django.db import models
  import uuid

  class Order(models.Model):
      id = models.UUIDField(primary_key=True, default=uuid.uuid4)
      user_id = models.CharField(max_length=50, null=True)
      guest_email = models.EmailField(null=True)
      guest_address = models.JSONField(null=True)
      items = models.JSONField()
      status = models.CharField(max_length=20)
      substitution_prefs = models.JSONField(default=dict)
      promotion = models.ForeignKey('product.Promotion', null=True, on_delete=models.SET_NULL)
      discount_applied = models.DecimalField(max_digits=10, decimal_places=2, default=0)
      coupon_code = models.CharField(max_length=20, null=True)
      created_at = models.DateTimeField(auto_now_add=True)
      delivery_instructions = models.TextField(blank=True)
      return_requested = models.BooleanField(default=False)
      return_status = models.CharField(max_length=20, null=True)
      return_reason = models.TextField(blank=True)

  class Subscription(models.Model):
      id = models.UUIDField(primary_key=True, default=uuid.uuid4)
      user_id = models.CharField(max_length=50)
      items = models.JSONField()
      frequency = models.CharField(max_length=20, choices=[('weekly', 'Weekly'), ('monthly', 'Monthly')])
      next_delivery = models.DateTimeField()
      status = models.CharField(max_length=20, default='active')
  ```
  ```python
  # services/order/order/services.py
  import requests
  from django.db import transaction
  from django.utils import timezone
  from datetime import timedelta
  from .models import Order, Subscription

  def process_guest_checkout(guest_email, guest_address, items, substitution_prefs, coupon_code=None):
      guest_id = f"guest_{uuid.uuid4().hex[:8]}"
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
              total_discount += max_discount
              item['promotion_id'] = applicable_promotion['id'] if applicable_promotion else None
          order = Order.objects.create(
              guest_email=guest_email,
              guest_address=guest_address,
              items=items,
              substitution_prefs=substitution_prefs,
              promotion_id=applicable_promotion['id'] if applicable_promotion else None,
              discount_applied=total_discount,
              coupon_code=coupon_code
          )
          requests.post('http://inventory:3000/stock/update', json=items)
          return order

  def request_return(order_id, reason):
      order = Order.objects.get(id=order_id)
      if timezone.now() > order.created_at + timedelta(days=7):
          raise ValueError('Return period expired')
      order.return_requested = True
      order.return_status = 'pending'
      order.return_reason = reason
      order.save()
      requests.post('http://payment:3000/payments/{order_id}/refund', json={'amount': order.discount_applied})
      requests.post('http://inventory:3000/stock/batch', json=order.items)
      return order
  ```
- **User (Wishlist and Loyalty)**:
  ```python
  # services/user/user/models.py
  from django.contrib.postgres.fields import ArrayField
  from django.db import models
  import uuid

  class User(models.Model):
      id = models.UUIDField(primary_key=True, default=uuid.uuid4)
      email = models.EmailField(unique=True, null=True)
      password = models.CharField(max_length=255, null=True)
      role = models.CharField(max_length=20, choices=[('customer', 'Customer'), ('admin', 'Admin'), ('staff', 'Staff')])
      name_en = models.CharField(max_length=255)
      name_vi = models.CharField(max_length=255)
      dietary_preferences = ArrayField(models.CharField(max_length=50), default=list)
      favorite_products = ArrayField(models.CharField(max_length=50), default=list)
      addresses = models.JSONField(default=list)
      loyalty_points = models.DecimalField(max_digits=10, decimal_places=2, default=0)
      social_provider = models.CharField(max_length=20, null=True)
      social_id = models.CharField(max_length=50, null=True)

  class LoyaltyTransaction(models.Model):
      id = models.UUIDField(primary_key=True, default=uuid.uuid4)
      user = models.ForeignKey('User', on_delete=models.CASCADE)
      points = models.DecimalField(max_digits=10, decimal_places=2)
      type = models.CharField(max_length=20, choices=[('earned', 'Earned'), ('redeemed', 'Redeemed')])
      order = models.ForeignKey('order.Order', null=True, on_delete=models.SET_NULL)
      created_at = models.DateTimeField(auto_now_add=True)
  ```
- **Analytics (Metabase Dashboard)**:
  ```typescript
  // services/analytics/src/analytics.controller.ts
  import { Controller, Get } from '@nestjs/common';
  import { InjectRepository } from '@nestjs/typeorm';
  import { Repository } from 'typeorm';

  @Controller('analytics')
  export class AnalyticsController {
    constructor(
      @InjectRepository(Order) private orderRepository: Repository<Order>,
    ) {}

    @Get('sales')
    async getSales() {
      return this.orderRepository
        .createQueryBuilder('order')
        .select('SUM(order.discount_applied)', 'revenue')
        .where('order.created_at > :date', { date: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) })
        .getRawOne();
    }
  }
  ```
- **Live Chat (Zendesk Integration)**:
  ```typescript
  // services/live-chat/src/support.controller.ts
  import { Controller, Post, Get, Body } from '@nestjs/common';
  import { ZendeskService } from './zendesk.service';

  @Controller('support')
  export class SupportController {
    constructor(private readonly zendeskService: ZendeskService) {}

    @Post('tickets')
    async createTicket(@Body() body: { user_id: string, order_id: string, issue: string }) {
      const ticket = await this.zendeskService.createTicket(body);
      return { ticket_id: ticket.id, status: 'open' };
    }
  }
  ```

**Automation Scripts**:
- **Seed Reviews**:
  ```python
  # scripts/seed_reviews.py
  from product.models import Product, Review
  from user.models import User
  from faker import Faker
  fake = Faker()

  products = Product.objects.all()[:100]
  users = User.objects.all()[:10]
  for product in products:
      for user in users:
          Review.objects.create(
              product=product,
              user=user,
              rating=fake.random_int(1, 5),
              comment=fake.sentence()
          )
  ```
- **Seed Bundles**:
  ```python
  # scripts/seed_bundles.py
  from product.models import Bundle
  from faker import Faker
  fake = Faker()

  for _ in range(50):
      Bundle.objects.create(
          name_en=fake.word(),
          name_vi=fake.word(),
          description=fake.paragraph(),
          included_products=[f"SKU{fake.random_int(1, 50000):06d}" for _ in range(3)],
          price=fake.random_int(10, 100)
      )
  ```

**Resource Considerations**:
- **Laptop Specs**: Allocate ~12GB RAM for containers (512MB each for Product Catalog, Inventory, Order, Delivery, Recommendation, Analytics; 256MB for Payment, User, Notification, Live Chat; 1GB for Elasticsearch; 512MB for Kafka; 2GB for dev container/`kind`).
- **Optimization**: Stop unused containers (`docker stop recommendation`) during development. Use `docker stats` to monitor CPU/RAM. Run Elasticsearch with minimal settings (`-Xms512m -Xmx1g`).
- **Constraints**: Ensure Docker supports ~14 containers. Use WSL2 (Windows) for performance if needed.

**Output**: Comprehensive microservices with scalable REST APIs, Kafka-driven events, Redis caching, and grocery-specific features (perishable tracking, substitutions, same-day slots, promotions, advanced search, recommendations, reviews, wishlists, returns, loyalty, subscriptions, analytics, live chat). Docker images deployed to `kind` via Helm, integrated with CI/CD, tested for ≥85% coverage, ready for Phase 3.

### Phase 3: Frontend and UX (4–5 Weeks)
**Goal**: Develop a mobile-first frontend with grocery-specific UX (search, checkout, delivery slots, promotions, recommendations, reviews, wishlists, order history, social sharing).

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 3.1 Prototype | Design UX in Figma for search, cart, checkout, slots, promotions, recommendations, reviews, wishlists, order history, social sharing. | 1 week | None | Mobile-first, multilingual (English/Vietnamese). Single-page checkout, calendar slot picker, promotion banners, recommendation widgets, review forms, wishlist buttons, order history page, social share icons. |
| 3.2 Build frontend | Develop React frontend, connect to GraphQL/REST APIs. | 2.5 weeks | 2.1–2.11, 3.1 | Use Apollo Client for GraphQL. Components: `ProductList`, `Cart`, `SlotPicker`, `PromotionBanner`, `RecommendationWidget`, `ReviewForm`, `WishlistButton`, `OrderHistory`, `SocialShare`. Cache with Apollo. Integrate Zendesk Chat widget. |
| 3.3 Multilingual | Implement English/Vietnamese toggle. | 3 days | 3.2 | Use `react-i18next`. Fetch translations from Product Catalog API. |
| 3.4 Social Sharing | Enable product sharing on social media (e.g., Facebook, Twitter). | 3 days | 3.2 | Use `react-share` for share buttons on product pages. Generate shareable links via Product Catalog API (`/products/{sku}`). |
| 3.5 Optimize | Ensure ≤2s page loads with Lighthouse. | 3 days | 3.2 | Compress images, use Cloudflare CDN, lazy-load products/recommendations/reviews. |

**Grocery Focus**: Streamlined checkout with substitution/promotion options, dietary/perishable/rating filters, popular items, recommendations, fast slot selection (≤500ms), review submission, wishlist management, order history access, social sharing for specialty products.
**Code Snippet (GraphQL Query)**:
```graphql
query GetProducts($category: String, $dietary: [String], $expiresSoon: Boolean, $minRating: Float) {
  products(category: $category, dietary: $dietary, expiresSoon: $expiresSoon, minRating: $minRating) {
    sku
    nameEn
    nameVi
    price
    isPerishable
    dietary
    averageRating
    reviews {
      rating
      comment
    }
  }
}
query GetWishlist($userId: String!) {
  wishlist(userId: $userId) {
    skus
  }
}
query GetOrderHistory($userId: String!, $guestEmail: String) {
  orders(userId: $userId, guestEmail: $guestEmail) {
    id
    items
    status
    createdAt
  }
}
```
**Output**: Responsive frontend integrated with backend, supporting all features, including reviews, wishlists, order history, and social sharing.

### Phase 4: Third-Party Integrations (2–3 Weeks)
**Goal**: Finalize Stripe, MoMo, Grab, ViettelPost, Gojek, Zendesk, and social login integrations.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 4.1 Stripe/MoMo | Complete Stripe.js/MoMo SDK integration, test webhooks, including subscription billing. | 1 week | 2.4 | NestJS, `@stripe/stripe-js`, MoMo SDK. Kafka topic: `payment_events`. Test with sandboxes. |
| 4.2 Grab/ViettelPost/Gojek | Integrate delivery APIs, test slot booking and return logistics. | 1 week | 2.6 | NestJS, REST APIs. Cache slots in Redis. Test with provider sandboxes. |
| 4.3 Social Login | Integrate Google/Facebook login via Firebase Authentication. | 3 days | 2.5 | Use `django-allauth` with Firebase SDK. Store `social_provider`, `social_id` in User model. |
| 4.4 Zendesk | Complete Zendesk Chat integration, test ticket creation. | 2 days | 2.11 | NestJS, Zendesk SDK. Sync tickets to Firestore. Test with sandbox. |
| 4.5 Extensibility | Abstract Payment/Delivery/Support Services for future providers. | 3 days | 4.1–4.4 | Strategy pattern in NestJS: `PaymentProcessor`, `DeliveryProvider`, `SupportProvider`. Store configs in Vault. |

**Grocery Focus**: Ensure ≤1s payment processing, ≤500ms slot updates, reliable webhook handling, seamless social login, responsive live chat.
**Output**: Fully integrated third-party services, extensible for future providers.

### Phase 5: Testing and Validation (3–4 Weeks)
**Goal**: Ensure reliability, performance, and grocery-specific functionality through comprehensive testing.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 5.1 Unit tests | Write tests for all services (≥80% coverage). | 1 week | 2.1–2.11 | pytest for Django, Jest for NestJS. Test promotions, search, recommendations, reviews, wishlists, returns, loyalty, subscriptions, analytics, live chat. |
| 5.2 Integration tests | Test service interactions (REST, Kafka). | 1 week | 2.1–2.11, 4.1–4.5 | Testcontainers for PostgreSQL/Firestore/Elasticsearch. Mock Stripe/MoMo/Grab/Zendesk APIs with `nock`. |
| 5.3 E2E tests | Test user flows (search, checkout, slots, recommendations, reviews, wishlists, order history, returns, subscriptions, live chat). | 1 week | 3.2, 5.2 | Cypress for browser tests. Test multilingual UX, substitutions, promotions, guest checkout, social sharing. |
| 5.4 Load tests | Simulate 300,000 users, validate performance. | 3 days | 5.2 | Locust: Test ≤500ms search, ≤1s checkout, ≤500ms live chat. Monitor with Prometheus/Grafana. |
| 5.5 Security tests | Scan for vulnerabilities, test JWT/RBAC/social login. | 2 days | 5.2 | OWASP ZAP for APIs, Snyk for dependencies. Test OAuth2 flows. |

**Grocery Focus**: Test inventory accuracy (≥99%), slot booking success (≥98%), promotion/recommendation accuracy, return processing, subscription reliability, live chat responsiveness under load.
**Output**: Fully tested platform meeting performance and reliability goals.

### Phase 6: Deployment and Beta Launch (2–3 Weeks)
**Goal**: Deploy the platform, conduct beta testing, and prepare for production.

| Task | Description | Duration | Dependencies | Technical Details |
|------|-------------|----------|--------------|-------------------|
| 6.1 Staging deploy | Deploy to staging Kubernetes cluster. | 3 days | 2.1–2.11, 3.2, 4.1–4.5 | Use Helm: `helm upgrade grocery ./charts`. Blue-green for Inventory/Delivery/Order. |
| 6.2 Beta testing | Launch to 100–500 users, collect feedback. | 1 week | 6.1, 5.1–5.5 | Use Django `waffle` for feature flags. Google Forms, Hotjar for feedback on reviews, wishlists, returns, subscriptions, live chat. |
| 6.3 Fix issues | Address beta feedback (e.g., UX, performance). | 1 week | 6.2 | Prioritize critical issues (e.g., slow checkout >1s, live chat delays). Update GitHub Issues. |
| 6.4 Production deploy | Deploy to production, monitor KPIs. | 3 days | 6.3 | Blue-green deployment, smoke tests (Postman). Monitor with Prometheus/Grafana/Metabase. |

**Grocery Focus**: Validate real-time inventory (≤100ms), delivery slots (≤500ms), promotion/recommendation UX, returns processing, subscription scheduling, live chat responsiveness during beta.
**Output**: Production-ready platform, initial KPIs (e.g., conversion rate ≥3%).

## Timeline
- **Total Duration**: 29–38 weeks (7–9 months) for all phases.
- **Breakdown**:
  - Phase 1: 1–2 weeks
  - Phase 2: 18–20 weeks
  - Phase 3: 4–5 weeks
  - Phase 4: 2–3 weeks
  - Phase 5: 3–4 weeks
  - Phase 6: 2–3 weeks
- **Assumption**: 20–30 hours/week, with flexibility for adjustments.

## Best Practices for Solo Development
- **Prioritization**: Focus on core services (Product Catalog, Inventory, Order, Payment, User, Delivery, Notification) first, then Search, Recommendation, Analytics, Live Chat, and additional integrations (MoMo/Gojek/Zendesk).
- **Automation**:
  - Use GitHub Actions for CI/CD (lint, test, deploy).
  - Deploy monitoring (Prometheus/Grafana, Loki) early.
  - Script tasks (e.g., `seed_promotions.py`, `seed_elasticsearch.js`, `seed_reviews.py`) in `/scripts`.
- **Task Management**: Use GitHub Projects with Kanban board. Label tasks (e.g., `priority:high`, `grocery`) and set milestones (“Phase 2”).
- **Technical Debt**:
  - Allocate 10% of time to refactoring (e.g., optimize Elasticsearch queries).
  - Use Snyk to catch vulnerabilities.
- **Grocery-Specific**:
  - Test inventory locking, substitutions, promotions, returns, subscriptions rigorously.
  - Cache delivery slots (Redis) for ≤500ms responses.
  - Validate multilingual UX, search, recommendations, reviews, wishlists, order history, live chat in beta.
- **Cloud Prep**: Use cloud-agnostic tools (MinIO, Helm) and test staging for AWS/GCP.

## Monitoring and Validation
- **KPIs** (Phase 6):
  - Conversion Rate: ≥3%.
  - Order Fulfillment: ≥95%.
  - Inventory Accuracy: ≥99%.
  - Slot Booking Success: ≥98%.
  - Uptime: ≥99.9%.
  - Recommendation Click-Through: ≥5%.
  - Review Submission Rate: ≥2% of orders.
  - Loyalty Point Redemption Rate: ≥10% of eligible users.
  - Subscription Retention Rate: ≥80% after 3 months.
- **Tools**: Prometheus/Grafana for metrics, GA4 for behavior, Hotjar for UX, Firestore for events, Metabase for business analytics.
- **Feedback**: Collect beta feedback via Google Forms/Hotjar to refine UX, focusing on reviews, wishlists, returns, subscriptions, live chat.

## Risks and Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| Overrunning timeline | Delays launch | Automate tests, use GitHub Projects, prioritize core services. |
| Performance issues | Slow UX (>1s checkout) | Cache with Redis, optimize Elasticsearch, test with Locust. |
| Third-party failures | Payment/delivery/support disruptions | Use circuit breakers, mock APIs, cache responses. |
| Technical debt | Maintenance burden | Refactor incrementally, enforce ≥80% coverage, use Snyk. |
| Resource constraints | Slow development | Stop unused containers, optimize Elasticsearch RAM, use WSL2. |
