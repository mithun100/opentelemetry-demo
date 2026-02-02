# OpenTelemetry Demo - Architecture with Splunk Integration

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ASTRONOMY SHOP APPLICATION                       │
│                                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │Frontend  │  │  Cart    │  │Checkout  │  │ Payment  │  │ Shipping │ │
│  │(Node.js) │  │  (.NET)  │  │  (Go)    │  │(Node.js) │  │  (Rust)  │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘ │
│       │             │             │             │             │        │
│  ┌────┴─────────────┴─────────────┴─────────────┴─────────────┴──────┐ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────┐ │ │
│  │  │  Product │  │   Email  │  │   Quote  │  │    Ad    │  │ LLM │ │ │
│  │  │ Catalog  │  │  (Ruby)  │  │   (PHP)  │  │  (Java)  │  │(Py) │ │ │
│  │  │   (Go)   │  └──────────┘  └──────────┘  └──────────┘  └─────┘ │ │
│  │  └──────────┘                                                      │ │
│  │                       + 5 more services...                         │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  All services instrumented with OpenTelemetry SDKs                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ OTLP Protocol
                                    │ (Traces, Metrics, Logs)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    OPENTELEMETRY COLLECTOR                               │
│                    (Central Telemetry Hub)                               │
│                                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐       │
│  │ Receivers  │→ │ Processors │→ │ Exporters  │→ │ Destinations│      │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘       │
│       │               │                │                │               │
│   OTLP/gRPC      Sampling         Batching        Multiple              │
│   OTLP/HTTP      Filtering        Formatting      Backends              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
    ┌───────────────────┐  ┌──────────────┐  ┌──────────────────┐
    │   LOCAL TOOLS     │  │  SPLUNK APM  │  │  SPLUNK IM       │
    │   (Open Source)   │  │  (Cloud)     │  │  (Cloud)         │
    ├───────────────────┤  ├──────────────┤  ├──────────────────┤
    │                   │  │              │  │                  │
    │ 📊 Jaeger         │  │ 🔍 Traces    │  │ 📈 Metrics       │
    │   - Traces UI     │  │ 🔥 Service   │  │ 📊 Dashboards    │
    │   - Search        │  │    Map       │  │ 🚨 Alerts        │
    │                   │  │ 📊 Dashboard │  │ 🏗️  Infra Mon    │
    │ 📈 Prometheus     │  │ 🚨 Alerts    │  │                  │
    │   - Metrics DB    │  │ 🔬 Profiling │  │                  │
    │   - Queries       │  │              │  │                  │
    │                   │  │              │  │                  │
    │ 📉 Grafana        │  │              │  │                  │
    │   - Dashboards    │  │              │  │                  │
    │   - Visualization │  │              │  │                  │
    │                   │  │              │  │                  │
    │ 🔍 OpenSearch     │  │              │  │                  │
    │   - Logs          │  │              │  │                  │
    └───────────────────┘  └──────────────┘  └──────────────────┘

         FREE                    ENTERPRISE         ENTERPRISE
    (Demo & Dev)            (Production Ready)  (Production Ready)
```

## 🎯 Key Benefits of This Architecture

### 1. **Vendor Neutrality**

- ✅ Same telemetry data sent to multiple destinations
- ✅ No vendor lock-in
- ✅ Switch backends without code changes

### 2. **Polyglot Support**

- 10+ microservices in 8+ languages
- All instrumented with OpenTelemetry
- Consistent telemetry across all services

### 3. **Flexible Deployment**

- **Local Development**: Free tools (Jaeger, Grafana)
- **Production**: Enterprise tools (Splunk, Datadog, etc.)
- **Hybrid**: Both simultaneously!

## 🔧 Configuration

### Current Setup

- ✅ All services sending to OpenTelemetry Collector
- ✅ Data flowing to Jaeger + Grafana + Prometheus
- ✅ **NEW**: Also sending to Splunk Observability Cloud

### What Changed?

**ZERO application code changes!**

Only configuration:

1. Added Splunk credentials to `.env`
2. Added Splunk exporter config (10 lines)
3. Updated collector to load Splunk config

## 📊 Data Flow

### Traces

```
Services → OTel Collector → ┬→ Jaeger (Local UI)
                            └→ Splunk APM (Cloud)
```

### Metrics

```
Services → OTel Collector → ┬→ Prometheus → Grafana (Local)
                            └→ Splunk IM (Cloud)
```

### Logs

```
Services → OTel Collector → ┬→ OpenSearch (Local)
                            └→ Splunk Log Observer (Cloud)
```

## 🎪 Demo Flow

1. **Browse Shop** → Generate traffic
2. **View Jaeger** → See traces locally
3. **Open Splunk** → Same traces in cloud!
4. **Toggle Feature Flags** → Simulate errors
5. **Show Both Tools** → Identical data, different UIs

## 🌟 Technologies Showcased

### Languages

- Go, Java, Python, C#, Node.js, PHP, Rust, Ruby, Elixir

### OpenTelemetry Components

- SDKs for all languages
- Auto-instrumentation (Java, Python, Node.js)
- Manual instrumentation
- Semantic conventions
- Context propagation

### Observability Pillars

- 📊 **Traces**: Distributed request tracking
- 📈 **Metrics**: Performance & resource monitoring
- 📝 **Logs**: Event logging with trace correlation

### Infrastructure

- Docker Compose
- Kafka (async messaging)
- PostgreSQL (database)
- Redis/Valkey (caching)
- Nginx (reverse proxy)

## 🚀 Access Points

| Service        | Local URL                        | Purpose                 |
| -------------- | -------------------------------- | ----------------------- |
| Astronomy Shop | http://localhost:8080            | Main application        |
| Jaeger         | http://localhost:8080/jaeger/ui/ | Trace visualization     |
| Grafana        | http://localhost:8080/grafana/   | Metrics dashboards      |
| Feature Flags  | http://localhost:8080/feature/   | Control chaos scenarios |
| Load Generator | http://localhost:8080/loadgen/   | Traffic generation      |

**Splunk Access**: Log into your Splunk Observability Cloud account to see the same data!
