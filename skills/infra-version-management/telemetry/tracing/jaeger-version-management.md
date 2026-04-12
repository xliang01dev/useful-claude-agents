# Jaeger Version Management

**Supplement to:** [infra-version-management SKILL.md](SKILL.md)

Use this guide when managing Jaeger versions and upgrades. Jaeger v1→v2 is an **architectural rewrite** — not a simple upgrade.

---

## Quick Facts

| Version | Status | Architecture | Image |
|---------|--------|--------------|-------|
| 1.x | Legacy | Monolithic all-in-one | `jaegertracing/all-in-one:1.x` |
| 2.x | Current | OpenTelemetry Collector-based | `jaegertracing/jaeger:2.x` |

**Critical:** v1 and v2 use **different image names entirely**.

---

## Version Pinning Example

```yaml
# docker-compose.yml - Jaeger v2
jaeger:
  image: jaegertracing/jaeger:v2.1.0  # v2 image name
  environment:
    COLLECTOR_OTLP_ENABLED: "true"
  ports:
    - "16686:16686"  # UI
    - "4317:4317"    # OTLP gRPC receiver
    - "4318:4318"    # OTLP HTTP receiver
  volumes:
    - ./jaeger-config.yaml:/etc/jaeger/config.yaml
```

---

## Breaking Changes: v1 → v2

**Architectural rewrite — NOT incremental upgrade.**

### Image Name Change

```yaml
# ❌ v1 (deprecated Dec 2025)
jaeger:
  image: jaegertracing/all-in-one:latest

# ✅ v2 (current)
jaeger:
  image: jaegertracing/jaeger:2.1.0
```

### Configuration Format Change

v2 requires **YAML configuration file** (environment variables not sufficient):

```yaml
# jaeger-config.yaml - v2 format
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  otlp:

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp]
```

### Command-Line Flags

v1 flags don't work in v2:

```bash
# ❌ v1 flags (won't work in v2)
--log-level=error
--query.http.server.host-port=:3030

# ✅ v2: Use YAML config instead
# See: jaeger-config.yaml
```

### Storage Backends

v1 and v2 support different backends:

- **v1:** Cassandra, Elasticsearch, in-memory
- **v2:** Focused on OTLP Collector + backend (Tempo, Jaeger, etc.)

---

## Configuration Example: Single-Instance v2 with OTLP

```yaml
# jaeger-config.yaml - MVP single-instance setup
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318  # OTLP HTTP receiver

processors:
  batch:
    timeout: 10s
    send_batch_size: 512

exporters:
  otlp:
    endpoint: localhost:4317

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

---

## Docker Compose Example

```yaml
version: '3'
services:
  jaeger:
    image: jaegertracing/jaeger:v2.1.0
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
      COLLECTOR_OTLP_ENDPOINT: "0.0.0.0:4318"
    volumes:
      - ./jaeger-config.yaml:/etc/jaeger/config.yaml
    ports:
      - "16686:16686"  # UI
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
    command:
      - "--config=/etc/jaeger/config.yaml"

  app:
    build: .
    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4318
    depends_on:
      - jaeger
```

---

## Testing Before Deployment

```bash
# Validate config syntax
docker run --rm -v $(pwd):/etc/jaeger jaegertracing/jaeger:v2.1.0 \
  --config=/etc/jaeger/config.yaml \
  --dry-run

# Start locally
docker-compose up jaeger

# Wait for startup
sleep 10

# Check health
curl http://localhost:16686/

# Check receiver is listening
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[]}'
```

---

## Upgrade Path: v1.x → v2.x (architectural rewrite)

**This is NOT a simple version bump.** Plan carefully:

1. **Read breaking changes first** (you're doing this now ✓)
2. **Update application SDKs**
   - v1 apps use Jaeger exporter: `jaeger-client` SDK
   - v2 apps use OTLP exporter: OpenTelemetry SDK
3. **Create v2 config file** (`jaeger-config.yaml`)
4. **Test v2 locally** with updated app code
5. **Run v1 and v2 in parallel** during transition (if needed)
6. **Migrate applications** to OTLP exporter
7. **Deploy v2**
8. **Decommission v1**

---

## SDK / Exporter Changes Required

### Application Side (When upgrading to v2)

```python
# ❌ v1 style (Jaeger-specific exporter)
from jaeger_client import Config

config = Config(
    config={
        'sampler': {'type': 'const', 'param': 1},
        'logging': True,
    },
    service_name='my-app',
)

# ✅ v2 style (OTLP exporter)
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

otlp_exporter = OTLPSpanExporter(
    endpoint="http://localhost:4318"
)

trace.get_tracer_provider().add_span_processor(
    trace.BatchSpanProcessor(otlp_exporter)
)
```

---

## Common Issues

### "unknown flag --log-level=error"

**Cause:** Using v1 flags with v2

**Fix:** Use YAML config file instead of command-line flags

---

### "Connection refused" from app to Jaeger

**Cause:** OTLP receiver binds to `127.0.0.1:4318` (localhost only); Docker can't reach it

**Fix:**
```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318  # Bind to all interfaces
```

---

### App sends traces but Jaeger shows "No traces found"

**Cause:** App still using v1 Jaeger exporter; v2 doesn't receive it

**Fix:** Update app to use OTLP exporter targeting `http://jaeger:4318`

---

## Monitoring infrastructure-versions.md

```markdown
## Jaeger

- Version: 2.1.0
- Image: jaegertracing/jaeger:2.1.0  ⚠️ NOT all-in-one
- Release Date: 2024-12-01
- Breaking Changes: v1→v2 architectural rewrite; YAML config required; OTLP exporter required
- Receiver Port: 4318 (OTLP HTTP)
- UI Port: 16686
- Health Check: GET /
- Status: ✅ Verified

### Upgrade Notes
- v1.x: Image name `jaegertracing/all-in-one` (deprecated Dec 2025)
- v2.x: Image name `jaegertracing/jaeger` (different image)
- Migration: Requires application code changes (SDK upgrade, exporter change)
- v1 data: Not migrated; start fresh with v2
- Last upgraded: 2026-04-11 (new v2 deployment)
```

---

## References

- [Jaeger Official Docs](https://www.jaegertracing.io/docs/)
- [Jaeger Docker Images](https://hub.docker.com/r/jaegertracing/jaeger)
- [OpenTelemetry Collector Architecture](https://opentelemetry.io/docs/reference/specification/protocol/exporter/)
- [Jaeger v1 to v2 Migration](https://www.jaegertracing.io/docs/latest/deployment/#jaeger-v2-migration)
