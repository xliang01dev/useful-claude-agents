# Infrastructure Version Management

**Reference document for:** [SKILL.md](SKILL.md) - event-driven-architecture skill

Use this skill when adding infrastructure services (observability stack, message brokers, databases) or upgrading existing ones. This skill prevents version cascade failures by enforcing explicit version pinning, compatibility verification, and breaking change review *before* you start.

## When to Use

- Adding a new observability service (Jaeger, Loki, Prometheus, OpenTelemetry)
- Upgrading any infrastructure container version
- Integrating SDKs (OpenTelemetry, exporters, instrumentation)
- Troubleshooting version-mismatch errors
- Setting up docker-compose or Kubernetes manifests

## Pre-Service Checklist: Before First `docker-compose up`

Complete this before you add any service to your infrastructure:

### 1. Pin Versions Explicitly
```yaml
# ❌ DON'T
jaeger:
  image: jaegertracing/jaeger:latest  # Could be v1 or v2 tomorrow

# ✅ DO
jaeger:
  image: jaegertracing/jaeger:2.1.0  # Explicit version
  # With comment:
  # jaegertracing/jaeger v2.1.0 (2024-12-01)
  # Breaking changes from v1: architectural rewrite, YAML config only
  # See: sdlc/infrastructure-versions.md
```

**Why:** `latest` tag can be stale or cached. "Latest" from 6 months ago isn't the same as latest today.

**Action:** Create a spreadsheet or `sdlc/infrastructure-versions.md` listing:
```markdown
| Service | Version | Release Date | Notes |
|---------|---------|--------------|-------|
| Jaeger | 2.1.0 | 2024-12-01 | Moved to OTel Collector; YAML config required |
| Loki | 3.0.0 | 2024-10-15 | Breaking changes from v2; see migration guide |
| Prometheus | v2.45.0 | 2024-08-10 | Generally stable; check breaking changes |
| OpenTelemetry SDK | 0.44b0 | 2024-09-01 | Pin exporters to same minor version |
```

---

### 2. Identify Image vs Version Mismatch

**Common pitfall:**

```yaml
# Problem: jaegertracing/all-in-one is v1 (deprecated Dec 2025)
# But you expected v2
jaeger:
  image: jaegertracing/all-in-one:latest  # This is v1!
  
# Solution: Different image for v2
jaeger:
  image: jaegertracing/jaeger:2.0.0  # Different image name entirely
```

**Action:** For each service, verify:
- Which image name corresponds to which version? (sometimes v2 is a different image)
- Is the `latest` tag actually current? (pull and check, or read GitHub releases)
- When was the image last updated? (Docker Hub shows this)

**Deliverable:** In `sdlc/infrastructure-versions.md`:
```markdown
## Jaeger
- v1: `jaegertracing/all-in-one:1.x` (deprecated Dec 2025)
- v2: `jaegertracing/jaeger:2.x` (different image)
- Current: `jaegertracing/jaeger:2.1.0`
- Last pulled: [date]
```

---

### 3. Read Breaking Changes Before Upgrading

**If upgrading an existing service, read the migration guide first:**

```markdown
### Loki v2 → v3 Breaking Changes

Before upgrading from Loki v2 to v3, you must:
1. Read https://grafana.com/docs/loki/latest/release-notes/v3-breaking-changes
2. Update config:
   - Removed: enforce_metric_name, retention_deletes_enabled, shared_store
   - Changed: schema_config.store values
   - Moved: tsdb_shipper config location
3. Test config locally first
4. Only then update docker-compose
```

**Action:** Before pulling a new container version, spend 10 minutes reading the Breaking Changes section on GitHub releases.

**Red flags:**
- Major version bump (v1→v2, v2→v3)
- Any mention of "config migration" or "breaking changes"
- New architectural approach (Jaeger v1→v2 is OpenTelemetry Collector-based)

---

### 4. Verify OpenTelemetry Compatibility Matrix

**If using OpenTelemetry SDKs + exporters + instrumentation:**

**Language-independent principle:**
```
SDK, API, exporters, and instrumentation must have compatible versions.
Mixing arbitrary versions causes import errors and runtime failures.
Pin all OTel packages to the same minor version.
```

**What compatibility means:**
- SDK version must match API version (they're tightly coupled)
- Exporters must be compatible with SDK version
- Instrumentation libraries must be compatible with SDK version
- All should be released around the same time (from same OTel release)

**Action plan:**
1. Check compatibility table for your language:
   - Python: https://github.com/open-telemetry/opentelemetry-python/blob/main/CONTRIBUTING.md
   - Go: https://github.com/open-telemetry/opentelemetry-go/releases
   - JavaScript: https://github.com/open-telemetry/opentelemetry-js/releases
   - Java: https://github.com/open-telemetry/opentelemetry-java/releases

2. Pin all OTel packages to same minor version in your dependency file

3. Test in dev environment before committing

**Language-specific examples:**

Python (requirements.txt):
```txt
# All packages pinned to same minor version (0.44)
opentelemetry-api>=0.44b0,<0.45
opentelemetry-sdk>=0.44b0,<0.45
opentelemetry-exporter-otlp>=0.44b0,<0.45
opentelemetry-instrumentation-requests>=0.44b0,<0.45
opentelemetry-instrumentation-flask>=0.44b0,<0.45
```

JavaScript/Node.js (package.json):
```json
{
  "dependencies": {
    "@opentelemetry/api": "^1.1.0",
    "@opentelemetry/sdk-node": "^0.44.0",
    "@opentelemetry/exporter-otlp": "^0.44.0",
    "@opentelemetry/instrumentation": "^0.44.0",
    "@opentelemetry/instrumentation-http": "^0.44.0"
  }
}
```

Go (go.mod):
```go
require (
    go.opentelemetry.io/otel v1.20.0
    go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc v0.42.0
    go.opentelemetry.io/otel/exporters/otlp/otlptrace v1.20.0
    go.opentelemetry.io/otel/sdk v1.20.0
)
```

Java (pom.xml):
```xml
<properties>
  <opentelemetry.version>1.30.1</opentelemetry.version>
</properties>
<dependencies>
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>${opentelemetry.version}</version>
  </dependency>
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-sdk</artifactId>
    <version>${opentelemetry.version}</version>
  </dependency>
</dependencies>
```

**❌ Anti-patterns to avoid:**
- Mixing random versions of SDK, API, exporter (causes ModuleNotFoundError, import errors)
- Using Jaeger-specific exporter without checking Jaeger version compatibility
- Pinning only one package, leaving others unpinned (they may drift)

**✅ Best practice:**
- Use same minor version for all OTel packages
- Document why you chose this version (was it latest stable? does it support your Jaeger version?)
- Check compatibility matrix BEFORE upgrading anything
- Test all packages together (not piecemeal)

---

### 5. Use OTLP for Modern Stacks

**Guidance on exporter selection:**

```markdown
### Exporter Choice

❌ OLD APPROACH: Jaeger-specific exporter
   - opentelemetry-exporter-jaeger (deprecated)
   - Ties you to Jaeger v1 or specific v2 versions
   - Not forward-compatible

✅ NEW APPROACH: OTLP (OpenTelemetry Protocol) exporter
   - opentelemetry-exporter-otlp
   - Works with Jaeger v2, Tempo, Grafana Cloud
   - Standard protocol; more flexible
   - Supported by all modern observability backends
```

**Action:** If starting fresh, use OTLP. Migrating from Jaeger-specific exporter? Plan for compatibility testing.

---

### 6. Test Configuration Locally First

**Before pushing to remote (language/deployment agnostic):**

```
Step 1: Validate configuration syntax
  - Use your infrastructure tool to validate (no deployment)
  - Examples: docker-compose config, kubectl --dry-run, terraform validate
  - Should complete without errors

Step 2: Start services locally
  - Docker Compose: docker-compose up
  - Kubernetes: kubectl apply -f manifests/
  - Terraform: terraform apply
  - Watch startup logs for errors

Step 3: Health check (wait 30 seconds)
  - Check each service started successfully
  - Call health endpoints (if available)
  - Read initialization logs for warnings

Step 4: Verify connectivity
  - Can services reach each other?
  - Can clients connect to services?
  - Check network, DNS, port availability

Step 5: Only after health checks pass
  - Commit changes to repo
  - Push to remote
  - Deploy to staging/production
```

**Deployment-specific examples:**

Docker Compose:
```bash
# Validate syntax
docker-compose config > /dev/null || exit 1

# Start services
docker-compose up

# Monitor startup (in another terminal)
sleep 30
docker-compose ps
docker-compose logs jaeger | tail -50
docker-compose logs loki | tail -50

# Kill when done
docker-compose down
```

Kubernetes:
```bash
# Validate manifests
kubectl apply -f manifests/ --dry-run=client

# Deploy to minikube/local cluster
kubectl apply -f manifests/

# Check pod status
kubectl get pods
kubectl logs -f deployment/jaeger
kubectl logs -f deployment/loki

# Cleanup
kubectl delete -f manifests/
```

Terraform:
```bash
# Validate syntax
terraform validate

# Plan (preview changes, don't apply)
terraform plan

# Apply to local/dev environment
terraform apply

# Destroy when done
terraform destroy
```

**Action:** Add validation to your CI/CD or pre-commit hooks:

Pre-commit hook (all languages/tools):
```bash
#!/bin/bash
# Validate infrastructure configuration before commit

if [ -f docker-compose.yml ]; then
  docker-compose config > /dev/null || exit 1
fi

if [ -d k8s/ ]; then
  kubectl apply -f k8s/ --dry-run=client >/dev/null || exit 1
fi

if [ -f main.tf ]; then
  terraform validate . || exit 1
fi

exit 0
```

---

### 7. Single-Instance vs Distributed Configs

**Critical: Don't copy enterprise cluster configs for single-instance setups**

**Language-independent principle:**
```
Enterprise infrastructure software ships with multi-instance cluster configs by default.
Single-instance MVP needs much simpler configuration.
Copying enterprise config results in: permission errors, validation failures, complexity.

Solution: Find the "single-instance" or "standalone" example config for your software.
```

**Common single-instance simplifications (all services):**

```
Disable distributed features (only for multi-instance):
  ❌ High Availability / Replication (set replica_factor=1, disable HA)
  ❌ Distributed consensus (use in-memory kv store, not Consul/etcd)
  ❌ Write-ahead logging (disable WAL if you don't need durability)
  ❌ Peer discovery (disable gossip protocol, cluster ring)
  ❌ Sharding/distribution (single instance is single shard)

Verify paths are correct (avoid permission errors):
  ❌ Don't write to /var (usually requires root)
  ✅ Use mounted volumes: /data, /logs, /cache (you control permissions)
  ✅ Ensure volume is mounted in your deployment
  ✅ Ensure app user has write permissions
```

**Service-specific examples:**

Loki (log aggregation):
```yaml
# ❌ WRONG: Multi-instance config
distributor:
  ring:
    kvstore:
      store: consul  # Needs Consul cluster
ingester:
  lifecycler:
    ring:
      replication_factor: 3  # Needs 3 replicas
  wal:
    enabled: true  # Needs persistent storage

# ✅ RIGHT: Single-instance config
ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory  # No external discovery
      replication_factor: 1  # Single instance
  wal:
    enabled: false  # No durability needed for MVP
  chunk_encoding: snappy

compactor:
  working_directory: /loki/compactor  # On mounted volume, not /var
```

Prometheus (metrics):
```yaml
# ❌ WRONG: Cluster setup
remote_storage:
  url: http://thanos-receive:10901/api/v1/receive  # Needs Thanos
  queue_config:
    capacity: 10000  # Enterprise scaling

# ✅ RIGHT: Single-instance config
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

Jaeger v2 (tracing):
```yaml
# ❌ WRONG: Multi-instance OTel Collector config
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 512

exporters:
  jaeger:
    endpoint: jaeger-backend:14250

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger]

# ✅ RIGHT: Single-instance for MVP
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318  # Bind to all interfaces, not localhost

processors:
  batch:
    timeout: 10s
    send_batch_size: 512

exporters:
  otlp:  # Direct to backend, no intermediate collector

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

**Action:** Before writing any config:
1. Search official docs for "single-instance", "standalone", or "MVP" config
2. Don't copy enterprise example as starting point
3. Read which config sections are for HA/clustering and remove them
4. Verify all file paths use mounted volumes, not system paths (/var)
5. Test configuration syntax locally before deploying

---

## Troubleshooting: When Version Issues Strike

### Error: "ModuleNotFoundError: No module named 'deprecated'"

**Cause:** OpenTelemetry SDK/exporter version mismatch; transitive dependency missing

**Fix:**
```bash
# Check what each package requires
pip show opentelemetry-exporter-jaeger | grep Requires

# If missing 'deprecated' package, add it explicitly:
pip install deprecated

# Better: Pin exact versions in requirements.txt
```

**Lesson:** Transitive dependencies matter. When an error mentions a missing module, it's usually a version incompatibility.

---

### Error: "unknown flag --log-level=error"

**Cause:** Mixing Jaeger v1 and v2 config syntax

**Fix:**
```yaml
# ❌ v1 syntax (won't work in v2)
command:
  - --log-level=error
  - --query.http.server.host-port=:3030

# ✅ v2 syntax (YAML config required)
# See: https://opentelemetry.io/docs/instrumentation/go/exporters/jaeger/
```

**Lesson:** v2 is a complete rewrite. Read the v2 docs; v1 flags don't apply.

---

### Error: "Loki started but services can't connect (Connection refused)"

**Cause:** Default OTLP HTTP receiver binds to `127.0.0.1:4318` (localhost only). Services in Docker can't reach localhost.

**Fix:**
```yaml
# Jaeger config.yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318  # Bind to all interfaces, not localhost
```

**Lesson:** Defaults are often security-hardened (localhost-only). When config is complex, fix it incrementally. Don't remove config to "simplify"; that hides the actual problem.

---

### Error: "Too many validation errors in config"

**Approach: Fix one error at a time**

```bash
# DON'T: Try to fix 5+ errors simultaneously
# DO: Fix first error, verify service starts, fix next error

docker-compose up loki 2>&1 | grep -i "error:" | head -1
# Fix that error in loki-config.yaml
docker-compose down
docker-compose up loki
# If healthy, fix next error
```

**Lesson:** Infrastructure config errors cascade. Isolate each error; verify health between changes.

---

## Deliverable: Infrastructure Versions Document

Create `sdlc/infrastructure-versions.md` documenting:

```markdown
# Infrastructure Versions & Configuration

Last updated: [date]

## Services

| Service | Version | Image | Breaking Changes | Minimal Config |
|---------|---------|-------|------------------|---|
| Jaeger | 2.1.0 | jaegertracing/jaeger:2.1.0 | v1→v2 architectural rewrite; YAML required | [link to config] |
| Loki | 3.0.0 | grafana/loki:3.0.0 | v2→v3 removed fields | [link to config] |
| Prometheus | v2.45.0 | prom/prometheus:v2.45.0 | None in v2.x | [link to config] |

## OpenTelemetry Compatibility

SDK: 0.44b0
Exporters: 0.44b0
Instrumentation: 0.44b0

All within minor version 0.44; tested together.

## Known Issues & Mitigations

1. **Jaeger v2 OTLP binding**: Default listens on localhost:4318; must bind 0.0.0.0:4318 for Docker
2. **Loki WAL permissions**: Single-instance should disable WAL; working_directory must be mounted volume
3. **Prometheus logging**: Use `--log.level=error` in command, not environment variable

## Testing Checklist

- [ ] All versions pinned explicitly (no `latest` tags)
- [ ] Compatibility matrix verified (OTel SDK/exporter/instrumentation)
- [ ] Config syntax matches service version (e.g., Jaeger v2 YAML)
- [ ] docker-compose config validates (`docker-compose config > /dev/null`)
- [ ] Service starts and passes health check
- [ ] Logs show no initialization errors (first 30 seconds)
- [ ] Only then: commit to repo
```

---

## Summary: The Workflow

1. **Before adding service:**
   - Pin version explicitly
   - Read breaking changes for that version
   - Verify image name/version correspondence
   - Pin OpenTelemetry versions (if applicable)

2. **Before first deployment:**
   - Test docker-compose config locally
   - Watch service startup logs
   - Verify health endpoints
   - Document in infrastructure-versions.md

3. **When upgrading:**
   - Read migration guide for major version
   - Update all config, not just image tag
   - Test locally first
   - Verify health after upgrade

4. **When troubleshooting version errors:**
   - Check image:version correspondence
   - Verify SDK/exporter compatibility
   - Review Breaking Changes section for that version
   - Fix one error at a time; verify health between changes

**Key principle:** Explicitness and verification up-front saves hours of debugging later.

