# 🚀 Quick Setup Guide - Splunk Integration

## ✅ Configuration Complete!

I've added Splunk Observability Cloud integration to your demo. Here's what was configured:

### Files Modified/Created:

1. ✅ `src/otel-collector/otelcol-config-splunk.yml` - Splunk exporter config
2. ✅ `.env` - Added Splunk credentials placeholders
3. ✅ `docker-compose.yml` - Updated collector to load Splunk config
4. ✅ `ARCHITECTURE.md` - Complete architecture diagram

---

## 🔑 Step 1: Add Your Splunk Credentials

Edit the `.env` file and replace these values:

```bash
# Line 42-43 in .env
SPLUNK_ACCESS_TOKEN=YOUR_TOKEN_HERE     # ← Replace with your actual token
SPLUNK_REALM=us0                        # ← Replace with your realm
```

### How to Get Your Credentials:

**Access Token:**

1. Log into Splunk Observability Cloud
2. Go to **Settings** → **Access Tokens**
3. Create new token or copy existing one
4. Token format: `ABC123xyz...`

**Realm:**
Look at your Splunk URL:

- `https://app.us0.signalfx.com/` → Realm is `us0`
- `https://app.us1.signalfx.com/` → Realm is `us1`
- `https://app.eu0.signalfx.com/` → Realm is `eu0`
- `https://app.ap0.signalfx.com/` → Realm is `ap0`

---

## 🔄 Step 2: Restart the Demo

Stop and restart to apply changes:

```bash
# Stop current demo
docker compose down

# Start with Splunk integration
docker compose up -d

# Check if collector is healthy
docker compose logs otel-collector --tail=20
```

---

## ✅ Step 3: Verify Data in Splunk

1. **Open Splunk APM**: https://app.YOUR_REALM.signalfx.com/
2. Navigate to **APM** → **Explore**
3. You should see services appear within 1-2 minutes:
   - frontend
   - checkoutservice
   - cart
   - paymentservice
   - etc.

4. Navigate to **Infrastructure** → **Hosts**
5. You should see metrics from your Docker host

---

## 🎯 What You Can Show in Your Demo

### 1. **Local Tools (Open Source)**

- **Jaeger**: http://localhost:8080/jaeger/ui/
- **Grafana**: http://localhost:8080/grafana/
- Show traces and metrics locally

### 2. **Splunk Cloud (Enterprise)**

- Open Splunk Observability Cloud
- Show THE SAME traces and metrics
- Demonstrate: "Same data, different visualization"

### 3. **Key Demo Points**

- ✅ **No code changes** - Only configuration
- ✅ **Vendor neutral** - OpenTelemetry standard
- ✅ **Multi-destination** - Send to multiple backends simultaneously
- ✅ **Polyglot** - 10+ services in 8+ languages

---

## 🐛 Troubleshooting

### Collector Not Starting?

Check logs:

```bash
docker compose logs otel-collector
```

Common issues:

- ❌ Token not set: Make sure you replaced `YOUR_TOKEN_HERE`
- ❌ Wrong realm: Verify your realm matches your Splunk URL
- ❌ Token expired: Generate a new token in Splunk

### Data Not Showing in Splunk?

1. Wait 2-3 minutes for first data
2. Generate traffic: Browse http://localhost:8080
3. Check token permissions: Needs "Ingest" permission
4. Verify collector logs show successful exports

### Still Issues?

```bash
# Check if collector can reach Splunk
docker compose exec otel-collector curl -v https://ingest.YOUR_REALM.signalfx.com/v2/trace
```

---

## 📊 Architecture Overview

```
Application Services (10+)
          ↓
    OTLP Protocol
          ↓
  OpenTelemetry Collector
          ↓
    ┌─────┴─────┐
    ↓           ↓
  Local      Splunk
  Tools      Cloud
  (Free)   (Enterprise)
```

**Key Insight**: The collector acts as a router, sending the same telemetry to multiple destinations without any application changes!

---

## 🎪 Presentation Tips

### Opening

"Today I'll show you a production-like application with 10+ microservices in 8 different languages, and how OpenTelemetry provides observability across all of them."

### Demo Flow

1. Show the running shop → "Real e-commerce application"
2. Browse and checkout → Generate traces
3. Open Jaeger → "Local development view"
4. Open Splunk → "Same data in production tool"
5. Toggle feature flag → Simulate errors
6. Show error in both tools → "Consistent experience"

### Key Messages

- **OpenTelemetry is vendor-neutral** - Not locked to any tool
- **Zero code changes** - Only configuration
- **Future-proof** - Easy to add/change backends
- **Industry standard** - CNCF graduated project

---

## 🎓 Next Steps

After your demo:

- Explore Splunk dashboards
- Set up alerts
- Try different feature flags
- Show service map
- Demonstrate profiling (if available)

Good luck with your presentation! 🚀
