# OpenTelemetry Collector Diagnostics Cheatsheet

## 🔍 Complete Health Check (Run This First)

```bash
echo "=== Environment Variables ===" && \
docker inspect otel-collector --format='{{range .Config.Env}}{{println .}}{{end}}' | grep SPLUNK && \
echo -e "\n=== Splunk Exporters Status ===" && \
docker compose logs otel-collector | grep -iE "sapm|signalfx" | grep -iE "info|synchronized" | tail -5 && \
echo -e "\n=== Recent Errors ===" && \
docker compose logs otel-collector --since 5m | grep -iE "error.*splunk|error.*sapm|error.*signalfx" || echo "No errors found ✅"
```

---

## 📋 Step-by-Step Diagnostic Commands

### 1️⃣ Verify Configuration is Loaded

```bash
# Check if Splunk environment variables are in the container
docker inspect otel-collector --format='{{range .Config.Env}}{{println .}}{{end}}' | grep SPLUNK

# Expected output:
# SPLUNK_REALM=us1
# SPLUNK_ACCESS_TOKEN=your_token_here
```

**What this checks:** Environment variables from .env file reached the container

---

### 2️⃣ Check Exporter Initialization

```bash
# Look for Splunk exporter startup messages
docker compose logs otel-collector 2>&1 | grep -iE "sapm|signalfx"

# Look specifically for initialization
docker compose logs otel-collector | grep -iE "sapm|signalfx" | grep -i "info" | head -10
```

**Success indicators:**

- ✅ `"otelcol.component.id": "sapm"` with `"exporter"` and `"traces"`
- ✅ `"otelcol.component.id": "signalfx"` with `"exporter"` and `"metrics"`
- ✅ `"Preparing to sync host properties to host dimension"`
- ✅ `"Host metadata synchronized"`

---

### 3️⃣ Look for Errors

```bash
# Search for authentication or connection failures
docker compose logs otel-collector --since 5m 2>&1 | grep -iE "(sapm|signalfx).*error|failed.*splunk|401|403|unauthorized"

# Check for any errors in last 5 minutes
docker compose logs otel-collector --since 5m | grep -i error

# Check for data being dropped
docker compose logs otel-collector --since 5m | grep -i "dropping data"
```

**Bad patterns to watch for:**

- ❌ `401 Unauthorized` → Bad token
- ❌ `403 Forbidden` → Token lacks permissions
- ❌ `connection refused` → Wrong endpoint or network issue
- ❌ `timeout` → Network/firewall issue
- ❌ `Exporting failed. Dropping data` → Critical failure

---

### 4️⃣ Validate Token Externally

```bash
# Test the token against Splunk API directly
curl -s -o /dev/null -w "%{http_code}" \
  -H "X-SF-TOKEN: YOUR_TOKEN_HERE" \
  https://api.us1.signalfx.com/v2/organization

# For other realms, replace us1 with: us0, us2, eu0, ap0, etc.
```

**Response codes:**

- ✅ `200` → Token valid and authorized
- ❌ `401` → Token invalid or expired
- ❌ `403` → Token exists but lacks permissions
- ❌ `404` → Wrong realm/endpoint

---

### 5️⃣ Verify Data Flow

```bash
# Check if collector is receiving telemetry from services
docker compose logs otel-collector --since 2m | grep -E "Traces.*spans.*[1-9]"

# Check for metrics
docker compose logs otel-collector --since 2m | grep -E "Metrics.*data points"

# See real-time data flow
docker compose logs -f otel-collector | grep -E "Traces|Metrics"
```

**Success indicators:**

- ✅ `Traces ... "spans": 17` → Receiving traces
- ✅ `Metrics ... "data points": 394` → Receiving metrics

---

### 6️⃣ Check Specific Components

```bash
# See all loaded exporters
docker compose logs otel-collector | grep "otelcol.component.kind.*exporter" | grep "otelcol.component.id"

# Check trace exporters only
docker compose logs otel-collector | grep '"otelcol.signal": "traces"' | grep exporter

# Check metrics exporters only
docker compose logs otel-collector | grep '"otelcol.signal": "metrics"' | grep exporter

# Monitor specific exporter in real-time
docker compose logs -f otel-collector | grep signalfx
```

---

### 7️⃣ Generate Test Traffic

```bash
# Generate 10 requests to create traces
for i in {1..10}; do
  curl -s http://localhost:8080/ > /dev/null
  sleep 1
done
echo "Traffic generated!"

# Generate continuous traffic
watch -n 2 'curl -s http://localhost:8080/ > /dev/null'
```

---

## 🎯 Common Troubleshooting Scenarios

### Scenario 1: No Data in Jaeger or Splunk

```bash
# 1. Check if services are running
docker compose ps

# 2. Check if collector is receiving data
docker compose logs otel-collector --tail=50 | grep -E "Traces|Metrics"

# 3. Check for errors
docker compose logs otel-collector --tail=100 | grep -i error

# 4. Restart collector to reload config
docker compose restart otel-collector

# 5. Wait and check logs
sleep 10 && docker compose logs otel-collector --tail=20
```

---

### Scenario 2: Splunk Token Issues

```bash
# 1. Verify token in .env file
cat .env | grep SPLUNK

# 2. Check if token loaded in container
docker inspect otel-collector --format='{{range .Config.Env}}{{println .}}{{end}}' | grep SPLUNK

# 3. Test token externally
curl -s -o /dev/null -w "%{http_code}" \
  -H "X-SF-TOKEN: $(grep SPLUNK_ACCESS_TOKEN .env | cut -d= -f2)" \
  https://api.$(grep SPLUNK_REALM .env | cut -d= -f2).signalfx.com/v2/organization

# 4. Check for auth errors in logs
docker compose logs otel-collector --since 10m | grep -iE "401|403|unauthorized"
```

---

### Scenario 3: Configuration Not Loading

```bash
# 1. Check if config file exists
ls -la src/otel-collector/otelcol-config-splunk.yml

# 2. Verify config is mounted in container
docker compose exec otel-collector ls -la /etc/otelcol-config-splunk.yml

# 3. Check if config is being loaded
docker compose logs otel-collector | grep "config" | head -10

# 4. Verify environment variables
docker compose config | grep -A30 "otel-collector:" | grep -A20 environment

# 5. Full restart with config reload
docker compose down otel-collector
docker compose up -d otel-collector
docker compose logs -f otel-collector
```

---

## 📊 Log Analysis Patterns

### ✅ Success Patterns (What to Look For)

```bash
# Exporter initialization
"otelcol.component.id": "sapm" ... "info"
"otelcol.component.id": "signalfx" ... "info"

# Connection success
"Host metadata synchronized"
"Preparing to sync host properties"

# Data flowing
Traces ... "spans": [any number > 0]
Metrics ... "data points": [any number > 0]
```

### ❌ Failure Patterns (Red Flags)

```bash
# Authentication issues
"401 Unauthorized"
"403 Forbidden"
"invalid token"

# Connection issues
"connection refused"
"timeout"
"dial tcp"

# Data loss
"Exporting failed. Dropping data"
"Permanent error"
"not retryable error"
```

---

## 🔧 Advanced Debugging

### View Full Collector Configuration

```bash
# See the merged configuration
docker compose exec otel-collector cat /etc/otelcol-config-splunk.yml
```

### Monitor Network Connections

```bash
# See active connections from collector
docker exec otel-collector netstat -tn | grep ESTABLISHED

# Check DNS resolution
docker exec otel-collector nslookup ingest.us1.signalfx.com
```

### Check Resource Usage

```bash
# See collector memory/CPU usage
docker stats otel-collector --no-stream

# See if collector is being throttled
docker compose logs otel-collector | grep -i "memory"
```

### Check OpenSearch (Local Log Storage)

```bash
# Get the dynamic port (OpenSearch uses random port mapping)
OPENSEARCH_PORT=$(docker compose port opensearch 9200 | cut -d: -f2)

# Check OpenSearch health
curl -s "http://localhost:$OPENSEARCH_PORT/_cluster/health" | jq

# List all indices
curl -s "http://localhost:$OPENSEARCH_PORT/_cat/indices?v"

# Count total logs
curl -s "http://localhost:$OPENSEARCH_PORT/otel-logs-*/_count" | jq '.count'

# View recent logs (last 5)
curl -s "http://localhost:$OPENSEARCH_PORT/otel-logs-*/_search?size=5&sort=@timestamp:desc" | jq '.hits.hits[]._source | {time: .["@timestamp"], service: .resource.["service.name"], message: .body}'

# Search logs by service
curl -s "http://localhost:$OPENSEARCH_PORT/otel-logs-*/_search" \
  -H 'Content-Type: application/json' \
  -d '{"query":{"match":{"resource.service.name":"frontend"}},"size":5}' | jq '.hits.hits[]._source.body'

# Count logs per service
curl -s "http://localhost:$OPENSEARCH_PORT/otel-logs-*/_search?size=0" \
  -H 'Content-Type: application/json' \
  -d '{"aggs":{"services":{"terms":{"field":"resource.service.name","size":30}}}}' | jq '.aggregations.services.buckets[] | {service: .key, count: .doc_count}'

# Check OpenSearch container status
docker compose ps opensearch

# View OpenSearch logs
docker compose logs opensearch --tail=20
```

**Why port is different:**

- Inside Docker network: `opensearch:9200` (container-to-container)
- From your host: `localhost:RANDOM_PORT` (dynamic port mapping)
- Always get the port first using `docker compose port opensearch 9200`

---

## 🎓 Understanding Log Structure

Every OpenTelemetry Collector log follows this pattern:

```
[container] | [timestamp] [level] [source_file] [message] {json_metadata}
```

**Example:**

```
otel-collector | 2026-01-26T02:43:36.449Z info hostmetadata/metadata.go:70
Host metadata synchronized {"otelcol.component.id": "signalfx"}
```

**Breaking it down:**

- `otel-collector` → Container name
- `2026-01-26T02:43:36.449Z` → UTC timestamp
- `info` → Log level (info, warn, error)
- `hostmetadata/metadata.go:70` → Source code location
- `Host metadata synchronized` → Human-readable message
- `{...}` → Structured JSON with component details

**Key JSON fields:**

- `otelcol.component.id` → Component name (sapm, signalfx, etc.)
- `otelcol.component.kind` → Type (exporter, receiver, processor)
- `otelcol.signal` → Data type (traces, metrics, logs)

---

## 🚀 Quick Commands Reference

```bash
# Service status
docker compose ps

# All services status
docker compose ps --format "table {{.Name}}\t{{.Status}}"

# Collector logs (last 50 lines)
docker compose logs otel-collector --tail=50

# Collector logs (last 5 minutes)
docker compose logs otel-collector --since 5m

# Follow collector logs in real-time
docker compose logs -f otel-collector

# Restart collector only
docker compose restart otel-collector

# Stop and start collector
docker compose down otel-collector && docker compose up -d otel-collector

# Check environment variables
docker inspect otel-collector --format='{{range .Config.Env}}{{println .}}{{end}}'

# Exec into collector container
docker compose exec otel-collector sh
```

---

## 🌐 Access Points

**Local Observability Tools:**

- Frontend: http://localhost:8080
- Jaeger UI: http://localhost:8080/jaeger/ui/
- Grafana: http://localhost:8080/grafana/ (admin/admin)
- Feature Flags: http://localhost:8080/feature/

**Splunk Observability Cloud:**

- US1: https://app.us1.signalfx.com/
- US0: https://app.us0.signalfx.com/
- EU0: https://app.eu0.signalfx.com/
- AP0: https://app.ap0.signalfx.com/

---

## 📝 Useful Grep Patterns

```bash
# Find all errors
docker compose logs otel-collector | grep -i error

# Find specific component errors
docker compose logs otel-collector | grep "signalfx" | grep error

# Find authentication issues
docker compose logs otel-collector | grep -iE "401|403|unauthorized|forbidden"

# Find connection issues
docker compose logs otel-collector | grep -iE "connection refused|timeout|dial tcp"

# Find data drops
docker compose logs otel-collector | grep -i "dropping data"

# Find specific signal type
docker compose logs otel-collector | grep '"otelcol.signal": "traces"'

# Find all exporters
docker compose logs otel-collector | grep '"otelcol.component.kind": "exporter"'

# Case-insensitive search with context
docker compose logs otel-collector | grep -i -A5 -B5 "pattern"
```

---

## 💡 Best Practices

1. **Always check logs with time filters** (`--since 5m`) to reduce noise
2. **Use `grep` patterns** to focus on relevant components
3. **Test token separately** before troubleshooting collector
4. **Generate traffic** before checking for data
5. **Wait 1-2 minutes** for first data to appear in Splunk
6. **Check for errors immediately** after configuration changes
7. **Monitor in real-time** (`-f` flag) when debugging
8. **Keep this cheatsheet** updated with your own findings!

---

**Last Updated:** January 26, 2026  
**Demo Version:** OpenTelemetry Demo 2.2.0  
**Collector Version:** 0.142.0
