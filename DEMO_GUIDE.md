# OpenTelemetry Astronomy Shop - Complete Demo Guide

## 📖 Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Application Architecture](#application-architecture)
- [All Endpoints](#all-endpoints)
- [Feature Flags](#feature-flags)
- [Demo Scenarios](#demo-scenarios)
- [Troubleshooting](#troubleshooting)

---

## Overview

**OpenTelemetry Astronomy Shop** is a microservices-based e-commerce demo application designed to showcase observability with OpenTelemetry across multiple programming languages and frameworks.

### What Makes This Demo Special?

- **27 Microservices** in **8+ Programming Languages**
- **Polyglot observability** - Go, Java, Python, C#, JavaScript, TypeScript, C++, Rust, Ruby, PHP, Kotlin, Elixir
- **Dual-destination telemetry** - Data flows to both local tools (Jaeger, Grafana, Prometheus) AND Splunk Observability Cloud
- **Real-world complexity** - Synchronous (gRPC/HTTP) + Asynchronous (Kafka) communication
- **Feature flags** - Control failures without redeployment
- **Vendor-neutral** - OpenTelemetry standard (OTLP) works with any observability backend

---

## Quick Start

### Prerequisites

- Docker Desktop installed and running
- At least 6GB RAM allocated to Docker
- Ports 8080 available

### Start the Application

```bash
# Navigate to the project directory
cd /Users/mithunb/Drive/observability/opentelemetry-demo

# Start all services
docker compose up -d

# Check all services are running (should see 28 containers)
docker compose ps

# Stop the application
docker compose down
```

### Access the Application

Once started, open your browser to: **http://localhost:8080**

---

## Application Architecture

### Business Domain: E-Commerce Astronomy Shop

The application simulates a working e-commerce website selling space-themed products (telescopes, satellites, etc.).

### Service Breakdown

#### 🛍️ **Core Business Services (10)**

| Service             | Language             | Port    | Purpose                          |
| ------------------- | -------------------- | ------- | -------------------------------- |
| **frontend**        | TypeScript (Next.js) | 8081    | Web UI for customers             |
| **frontend-proxy**  | Go (Envoy)           | 8080    | Entry point, routes all traffic  |
| **ad**              | Java (Spring Boot)   | Dynamic | Contextual advertisements        |
| **cart**            | C# (.NET)            | Dynamic | Shopping cart management         |
| **checkout**        | Go                   | Dynamic | Order processing orchestrator    |
| **product-catalog** | Go                   | Dynamic | Product database & search        |
| **product-reviews** | Python (Flask)       | Dynamic | Product reviews & AI assistant   |
| **recommendation**  | Python               | Dynamic | ML-based product recommendations |
| **currency**        | C++                  | Dynamic | Multi-currency conversion        |
| **shipping**        | Rust                 | Dynamic | Shipping quotes & fulfillment    |

#### 💰 **Transactional Services (3)**

| Service        | Language             | Purpose                   |
| -------------- | -------------------- | ------------------------- |
| **payment**    | JavaScript (Node.js) | Payment processing        |
| **email**      | Ruby                 | Order confirmation emails |
| **accounting** | C# (.NET)            | Financial record keeping  |

#### 🔧 **Supporting Services (3)**

| Service             | Language | Purpose                     |
| ------------------- | -------- | --------------------------- |
| **fraud-detection** | Kotlin   | Fraud analysis via Kafka    |
| **quote**           | PHP      | Shipping price calculations |
| **llm**             | Python   | AI assistant for products   |

#### 🎮 **Management & Testing (4)**

| Service            | Technology       | Purpose                           |
| ------------------ | ---------------- | --------------------------------- |
| **flagd**          | Go               | Feature flag server (OpenFeature) |
| **flagd-ui**       | Elixir (Phoenix) | Feature flag management UI        |
| **load-generator** | Python (Locust)  | Synthetic traffic generation      |
| **image-provider** | Go               | Product image hosting             |

#### 🗄️ **Infrastructure (4)**

| Service            | Purpose                         |
| ------------------ | ------------------------------- |
| **kafka**          | Message broker for async events |
| **postgresql**     | Database for accounting service |
| **valkey-cart**    | Redis-compatible cache for cart |
| **otel-collector** | OpenTelemetry data pipeline     |

#### 📊 **Observability Stack (4)**

| Service        | Purpose                         |
| -------------- | ------------------------------- |
| **jaeger**     | Distributed trace visualization |
| **grafana**    | Metrics & log dashboards        |
| **prometheus** | Time-series metrics database    |
| **opensearch** | Log storage & search            |

---

### Communication Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      User Browser                           │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP :8080
┌────────────────────────▼────────────────────────────────────┐
│                   frontend-proxy (Envoy)                    │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP :8081
┌────────────────────────▼────────────────────────────────────┐
│                  frontend (Next.js)                         │
│                Browser Instrumentation                      │
└──────┬──────────┬──────────┬──────────┬─────────────────────┘
       │          │          │          │
       │ gRPC     │ gRPC     │ gRPC     │ gRPC
       ▼          ▼          ▼          ▼
┌─────────┐ ┌────────┐ ┌──────────┐ ┌────────┐
│ product │ │  cart  │ │checkout  │ │  ad    │
│ catalog │ │        │ │          │ │        │
└─────────┘ └───┬────┘ └────┬─────┘ └────────┘
                │           │
                │           │ Orchestrates 6+ services
                ▼           ▼
         ┌──────────┐  ┌──────────┐
         │  valkey  │  │ payment  │
         │  (Redis) │  │ currency │
         └──────────┘  │ shipping │
                       │ email    │
                       │ cart     │
                       └────┬─────┘
                            │ Kafka
                       ┌────▼─────┐
                       │  kafka   │
                       └────┬─────┘
                            │
                ┌───────────┼───────────┐
                ▼                       ▼
         ┌──────────┐           ┌──────────┐
         │accounting│           │  fraud-  │
         │          │           │detection │
         └────┬─────┘           └──────────┘
              │
              ▼
       ┌──────────┐
       │postgresql│
       └──────────┘
```

### Key User Journeys

#### 1️⃣ **Browse Products**

```
User → frontend-proxy → frontend → [product-catalog, currency, recommendation, ad]
Services: 5 | Protocols: HTTP, gRPC | Spans: ~6
```

#### 2️⃣ **Add to Cart**

```
User → frontend → cart → valkey-cart (Redis)
Services: 3 | Data Store: Redis | Spans: ~3
```

#### 3️⃣ **Complete Checkout** (Most Complex!)

```
User → frontend → checkout orchestrates:
  ├→ cart (GetCart)
  ├→ product-catalog (GetProduct for each item)
  ├→ currency (Convert)
  ├→ shipping → quote (GetQuote)
  ├→ payment (Charge)
  ├→ shipping (ShipOrder)
  ├→ email (SendOrderConfirmation)
  ├→ kafka (Publish OrderPlaced event)
  └→ cart (EmptyCart)

Async processing:
  - accounting ← kafka → postgresql
  - fraud-detection ← kafka

Services: 11+ | Duration: ~500ms-2s | Spans: 20+
```

---

## All Endpoints

### 🛍️ **Application Endpoints**

| Endpoint             | URL                           | Description                                 |
| -------------------- | ----------------------------- | ------------------------------------------- |
| **Main Store**       | http://localhost:8080         | E-commerce website (browse, cart, checkout) |
| **Feature Flags UI** | http://localhost:8080/feature | Toggle feature flags & simulate failures    |
| **Product Images**   | http://localhost:8080/images  | Static product images                       |

### 📊 **Local Observability Endpoints**

| Tool           | URL                              | Credentials   | Purpose                                                             |
| -------------- | -------------------------------- | ------------- | ------------------------------------------------------------------- |
| **Jaeger UI**  | http://localhost:8080/jaeger/ui  | None          | Distributed traces across all services                              |
| **Grafana**    | http://localhost:8080/grafana    | admin / admin | Pre-built dashboards for metrics                                    |
| **Prometheus** | http://localhost:8080/prometheus | None          | Raw metrics query (PromQL)                                          |
| **OpenSearch** | http://localhost:PORT            | admin / admin | Log storage (check port with `docker compose port opensearch 9200`) |

### ☁️ **Splunk Observability Cloud**

| Service            | URL                                       | Description                                    |
| ------------------ | ----------------------------------------- | ---------------------------------------------- |
| **APM**            | https://app.us1.signalfx.com/#/apm        | Application traces                             |
| **Infrastructure** | https://app.us1.signalfx.com/#/infra      | Host & service metrics                         |
| **Dashboards**     | https://app.us1.signalfx.com/#/dashboards | Custom dashboards                              |
| **RUM**            | https://app.us1.signalfx.com/#/rum        | Real User Monitoring (requires Splunk RUM SDK) |

### 🔧 **Backend Service Endpoints** (gRPC - Internal Only)

All backend services use **dynamically assigned ports**. Check with:

```bash
docker compose ps
```

Services: ad, cart, checkout, currency, email, payment, product-catalog, product-reviews, recommendation, shipping, quote

---

## Feature Flags

Feature flags allow you to **control application behavior in real-time** without redeployment. Perfect for demonstrating error handling and observability!

### Access Feature Flags

**URL:** http://localhost:8080/feature

### Available Feature Flags

| Flag Name                        | Default State | Effect When ENABLED                | Use Case                      |
| -------------------------------- | ------------- | ---------------------------------- | ----------------------------- |
| **productCatalogFailure**        | OFF           | Product list fails to load         | Demo error propagation        |
| **recommendationServiceFailure** | OFF           | Product suggestions disappear      | Demo graceful degradation     |
| **adServiceFailure**             | OFF           | Advertisements fail to display     | Demo non-critical failure     |
| **cartServiceFailure**           | OFF           | Cart operations fail               | Demo critical path failure    |
| **paymentServiceFailure**        | OFF           | Checkout payment fails             | Demo transaction failure      |
| **productCatalogCache**          | ON            | Enable Redis caching for products  | Demo performance optimization |
| **recommendationCache**          | ON            | Enable caching for recommendations | Demo caching strategy         |

### How to Use Feature Flags

1. **Open Feature Flags UI:** http://localhost:8080/feature
2. **Toggle a flag** (e.g., "adServiceFailure" to ON)
3. **Refresh the shop** at http://localhost:8080
4. **Observe the effect** (ads section shows error)
5. **Check traces in Jaeger** - you'll see error spans in red
6. **Toggle back OFF** - service recovers immediately

### Demo Scenarios with Feature Flags

#### Scenario 1: Non-Critical Failure (Graceful Degradation)

```
1. Enable "adServiceFailure"
2. Shop still works - only ads are missing
3. Trace shows error but page loads successfully
4. Demonstrates resilience
```

#### Scenario 2: Critical Path Failure

```
1. Enable "cartServiceFailure"
2. Cannot add items to cart
3. Trace shows failure in cart service
4. Demonstrates cascading failure detection
```

#### Scenario 3: Performance Comparison

```
1. Disable "productCatalogCache"
2. Browse products - note latency in traces
3. Enable cache
4. Browse again - see reduced latency
5. Demonstrates performance optimization visibility
```

---

## Demo Scenarios

### 🎬 **30-Minute Demo Flow for AppDynamics vs Splunk Presentation**

#### **Part 1: Application Overview (5 minutes)**

1. Show the working application at http://localhost:8080
2. Browse products, add to cart, complete a purchase
3. Explain: "This is a polyglot microservices app with 27 services in 8+ languages"

#### **Part 2: Distributed Tracing (10 minutes)**

1. Complete a checkout transaction
2. Open Jaeger: http://localhost:8080/jaeger/ui
3. Search for service "checkout" with recent traces
4. Click on a trace - show the waterfall view
5. **Key Points:**
   - Trace spans across 11+ services
   - Multiple languages (Go, Python, C#, Node.js, etc.)
   - Identify slowest service in the chain
   - Show error propagation (if any)

6. Open Splunk APM: https://app.us1.signalfx.com/#/apm
7. Show the **same trace** in Splunk
8. **Key Points:**
   - Vendor neutrality - same data, different UI
   - OpenTelemetry standard (OTLP)
   - No vendor lock-in

#### **Part 3: Feature Flags & Error Handling (8 minutes)**

1. Open Feature Flags UI: http://localhost:8080/feature
2. Enable "adServiceFailure"
3. Refresh shop - ads section shows error
4. Search in Jaeger for "frontend" service
5. Show error span in red
6. **Key Points:**
   - Graceful degradation (site still works)
   - Error visibility in traces
   - Quick recovery (toggle OFF)

#### **Part 4: Metrics & Dashboards (5 minutes)**

1. Open Grafana: http://localhost:8080/grafana (admin/admin)
2. Navigate to "Demo" folder dashboards
3. Show service-specific dashboards:
   - Request rates, error rates, latency (RED metrics)
   - Service dependencies
4. Open Splunk Infrastructure: https://app.us1.signalfx.com/#/infra
5. Show host metrics and service map

#### **Part 5: Polyglot & Vendor Neutrality (2 minutes)**

**Wrap-up message:**

- **27 services, 8+ languages** - OpenTelemetry works everywhere
- **Zero code changes** to switch vendors
- **Same data** flows to Jaeger AND Splunk simultaneously
- **Standards-based** approach (OTLP)
- Perfect for organizations evaluating APM vendors

---

## Troubleshooting

### Common Issues

#### ❌ Services showing "red" errors in logs

**Status:** Normal behavior
**Cause:** Load generator creates high traffic, otel-collector queue fills temporarily
**Impact:** None - services retry and succeed
**Action:** Ignore warnings like "sending queue is full" or "StatusCode.UNAVAILABLE"

#### ❌ Cannot access http://localhost:8080

**Check:**

```bash
# Verify all services are running
docker compose ps

# Check frontend-proxy is healthy
docker compose logs frontend-proxy --tail 20

# Restart if needed
docker compose restart frontend-proxy
```

#### ❌ Traces not appearing in Jaeger

**Check:**

```bash
# Verify otel-collector is running
docker compose logs otel-collector --tail 50

# Look for "Host metadata synchronized" - indicates success
```

#### ❌ Feature flags not loading

**Fix:**

```bash
# Restart flagd and flagd-ui
docker compose restart flagd flagd-ui

# Verify flagd is healthy
docker compose logs flagd --tail 20
```

### Useful Commands

```bash
# View logs for a specific service
docker compose logs -f [service-name]

# Restart a specific service
docker compose restart [service-name]

# Check service health
docker compose ps

# View all environment variables
docker compose config

# Stop all services
docker compose down

# Clean up everything (including volumes)
docker compose down -v

# Rebuild a specific service
docker compose build [service-name]
docker compose up -d [service-name]
```

### OpenSearch Port Discovery

OpenSearch uses dynamic port mapping on macOS:

```bash
# Find the mapped port
OPENSEARCH_PORT=$(docker compose port opensearch 9200 | cut -d: -f2)
echo "OpenSearch is accessible at: http://localhost:${OPENSEARCH_PORT}"

# Access OpenSearch
curl -u admin:admin -k "http://localhost:${OPENSEARCH_PORT}/_cat/indices"
```

---

## Additional Resources

### Configuration Files

- **Splunk Integration:** `src/otel-collector/otelcol-config-splunk.yml`
- **Environment Variables:** `.env`
- **Feature Flags:** `src/flagd/demo.flagd.json`
- **Diagnostics Guide:** `DIAGNOSTICS_CHEATSHEET.md`
- **Architecture:** `ARCHITECTURE.md`
- **Splunk Setup:** `SPLUNK_SETUP.md`

### Observability Data Flow

```
All Services (27)
    │
    ├─ Traces (OTLP/gRPC) ───┐
    ├─ Metrics (OTLP/gRPC) ──┼─→ otel-collector
    └─ Logs (OTLP/HTTP) ─────┘        │
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
            ┌──────────────┐   ┌─────────────┐  ┌──────────────┐
            │   Jaeger     │   │ Prometheus  │  │  OpenSearch  │
            │  (Traces)    │   │  (Metrics)  │  │   (Logs)     │
            └──────────────┘   └─────────────┘  └──────────────┘
                                       │
                                       ▼
                               ┌──────────────┐
                               │   Grafana    │
                               │ (Dashboards) │
                               └──────────────┘

                    PLUS:

                    otel-collector
                           │
                    ┌──────┼──────┐
                    │             │
                    ▼             ▼
            ┌──────────────┐  ┌──────────────┐
            │ Splunk APM   │  │  Splunk IM   │
            │  (Traces)    │  │  (Metrics)   │
            └──────────────┘  └──────────────┘
```

---

## Quick Reference Card

### 🚀 Start/Stop

```bash
docker compose up -d        # Start all services
docker compose down         # Stop all services
docker compose ps           # Check status
```

### 🌐 Key URLs

- **Shop:** http://localhost:8080
- **Jaeger:** http://localhost:8080/jaeger/ui
- **Grafana:** http://localhost:8080/grafana (admin/admin)
- **Feature Flags:** http://localhost:8080/feature
- **Splunk APM:** https://app.us1.signalfx.com/#/apm

### 🎛️ Demo Flow

1. Browse & checkout → Generate traces
2. View in Jaeger → Show distributed trace
3. View in Splunk APM → Same data, different vendor
4. Toggle feature flag → Create controlled failure
5. Observe error in traces → Show error handling

### 📊 Key Talking Points

- ✅ 27 microservices in 8+ languages
- ✅ Vendor-neutral OpenTelemetry standard
- ✅ Dual-destination telemetry (Jaeger + Splunk)
- ✅ Real-world complexity (sync + async)
- ✅ Feature flags for controlled failures
- ✅ Zero code changes to switch vendors

---

**Created:** February 2, 2026  
**Project:** OpenTelemetry Astronomy Shop Demo  
**Purpose:** AppDynamics vs Splunk Observability Comparison Presentation
