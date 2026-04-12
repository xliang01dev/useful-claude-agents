# OpenTelemetry Version Management

**Supplement to:** [infra-version-management SKILL.md](SKILL.md)

Use this guide when managing OpenTelemetry SDKs, exporters, and instrumentation versions. **Compatibility between components is critical** — version mismatches cascade quickly.

---

## Quick Facts

OpenTelemetry is a **multi-component system**. Each must have compatible versions:

- **API:** Core interfaces (stable, less frequent updates)
- **SDK:** Implementation (tracks with API)
- **Exporter:** Sends data to backends (must match SDK)
- **Instrumentation:** Hooks for libraries/frameworks (must match SDK/API)

---

## Critical Rule: Same Minor Version

**All OpenTelemetry packages must be pinned to the same minor version.**

```
✅ OK: All 0.44 (SDK 0.44, Exporter 0.44, Instrumentation 0.44)
❌ FAIL: SDK 0.44 + Exporter 0.43 + Instrumentation 0.45 (cascade failure)
```

---

## Version Pinning: By Language

### Python

```txt
# requirements.txt
opentelemetry-api==0.44b0
opentelemetry-sdk==0.44b0
opentelemetry-exporter-otlp==0.44b0
opentelemetry-instrumentation==0.44b0
opentelemetry-instrumentation-requests==0.44b0
opentelemetry-instrumentation-flask==0.44b0
```

### JavaScript/Node.js

```json
{
  "dependencies": {
    "@opentelemetry/api": "^1.1.0",
    "@opentelemetry/sdk-node": "^0.44.0",
    "@opentelemetry/sdk-trace-node": "^0.44.0",
    "@opentelemetry/exporter-otlp": "^0.44.0",
    "@opentelemetry/instrumentation": "^0.44.0",
    "@opentelemetry/instrumentation-http": "^0.44.0",
    "@opentelemetry/instrumentation-express": "^0.44.0"
  }
}
```

### Go

```go
// go.mod
require (
    go.opentelemetry.io/otel v1.20.0
    go.opentelemetry.io/otel/sdk v1.20.0
    go.opentelemetry.io/otel/exporter/otlp/otlptrace/otlptracehttp v1.20.0
    go.opentelemetry.io/otel/instrumentation v0.42.0
    go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp v0.42.0
)
```

### Java

```xml
<properties>
  <opentelemetry.version>1.30.1</opentelemetry.version>
  <opentelemetry-instrumentation.version>1.30.0</opentelemetry-instrumentation.version>
</properties>

<dependencies>
  <!-- API -->
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>${opentelemetry.version}</version>
  </dependency>

  <!-- SDK -->
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-sdk</artifactId>
    <version>${opentelemetry.version}</version>
  </dependency>

  <!-- Exporter -->
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
    <version>${opentelemetry.version}</version>
  </dependency>

  <!-- Instrumentation -->
  <dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-instrumentation-api</artifactId>
    <version>${opentelemetry-instrumentation.version}</version>
  </dependency>
</dependencies>
```

---

## Breaking Changes: Major Versions

### v0.x → v1.x (Stability Transition)

**API stabilized, SDK interface changes:**

```python
# ❌ v0.x style
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

# ✅ v1.x style (mostly compatible, minor changes)
from opentelemetry import trace
tracer = trace.get_tracer_provider().get_tracer(__name__)
```

### Exporter Changes

- **v0.x:** Separate Jaeger exporter (`opentelemetry-exporter-jaeger`)
- **v1.x:** Unified OTLP exporter (`opentelemetry-exporter-otlp`)

```python
# ❌ v0.x style (deprecated)
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
exporter = JaegerExporter(agent_host_name="localhost", agent_port=6831)

# ✅ v1.x style (current)
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
exporter = OTLPSpanExporter(endpoint="http://localhost:4318")
```

---

## Configuration Example: Python v0.44b0

```python
# main.py - OpenTelemetry v0.44b0 single-instance setup
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

# Set up exporter
otlp_exporter = OTLPSpanExporter(
    endpoint="http://localhost:4318/v1/traces"
)

# Set up tracer provider
tracer_provider = TracerProvider()
tracer_provider.add_span_processor(SimpleSpanProcessor(otlp_exporter))
trace.set_tracer_provider(tracer_provider)

# Get tracer
tracer = trace.get_tracer(__name__)

# Create spans
with tracer.start_as_current_span("my-span"):
    print("Span created and exported to http://localhost:4318")
```

---

## Testing Dependency Compatibility

### Python: Check Transitive Dependencies

```bash
# See what opentelemetry-exporter-otlp requires
pip show opentelemetry-exporter-otlp | grep Requires

# Install compatible versions
pip install \
  opentelemetry-api==0.44b0 \
  opentelemetry-sdk==0.44b0 \
  opentelemetry-exporter-otlp==0.44b0

# Verify all installed
pip list | grep opentelemetry
```

### JavaScript: Check npm

```bash
# Check package dependencies
npm list @opentelemetry/sdk-node

# Install compatible versions
npm install \
  @opentelemetry/api@^1.1.0 \
  @opentelemetry/sdk-node@^0.44.0 \
  @opentelemetry/exporter-otlp@^0.44.0
```

### Go: Check go.mod Compatibility

```bash
# List dependencies
go list -u all | grep opentelemetry

# Update to compatible version
go get -u go.opentelemetry.io/otel@v1.20.0
go get -u go.opentelemetry.io/otel/sdk@v1.20.0
go get -u go.opentelemetry.io/otel/exporter/otlp/otlptrace/otlptracehttp@v1.20.0
```

---

## Common Version Mismatch Errors

### "ModuleNotFoundError: No module named 'deprecated'"

**Cause:** Transitive dependency missing due to version mismatch

**Fix:**
```bash
# See what's required
pip show opentelemetry-exporter-otlp | grep Requires

# Pin all to same minor version
pip install opentelemetry-sdk==0.44b0 opentelemetry-exporter-otlp==0.44b0
```

---

### "TypeError: __init__() got an unexpected keyword argument 'endpoint'"

**Cause:** Exporter API changed between versions (SDK and exporter mismatch)

**Fix:**
```python
# Check which exporter you're using
import opentelemetry.exporter.otlp.proto.http.trace_exporter as exporter_module
print(exporter_module.__file__)

# Ensure SDK and exporter have same minor version
# pip install opentelemetry-sdk==0.44b0 opentelemetry-exporter-otlp==0.44b0
```

---

### "No spans exported" (Spans created but don't reach backend)

**Cause:** Exporter configured wrong or backend unreachable

**Fix:**
```python
# Check endpoint is correct
exporter = OTLPSpanExporter(endpoint="http://localhost:4318/v1/traces")

# Add debug logging
import logging
logging.basicConfig(level=logging.DEBUG)
logging.getLogger("opentelemetry").setLevel(logging.DEBUG)

# Test backend is reachable
import requests
requests.get("http://localhost:4318/-/health")  # Or backend's health endpoint
```

---

## Monitoring infrastructure-versions.md

```markdown
## OpenTelemetry

- API Version: 0.44b0
- SDK Version: 0.44b0
- Exporter (OTLP): 0.44b0
- Instrumentation: 0.44b0
- Backend: Jaeger v2.1.0 (http://localhost:4318)
- Status: ✅ Verified and compatible

### Compatibility Notes

**Critical:** All packages must be same minor version (0.44)

- api: 0.44b0
- sdk: 0.44b0
- exporter-otlp: 0.44b0
- instrumentation-requests: 0.44b0
- instrumentation-flask: 0.44b0

### Verified Compatibility

- ✅ Python SDK 0.44b0 + OTLP Exporter 0.44b0 + Jaeger 2.1.0
- ✅ Flask instrumentation 0.44b0
- ✅ Requests instrumentation 0.44b0

### Version Mismatch Prevention

- Lock all versions in requirements.txt (no ^, ~, or >=)
- Update all together (never piecemeal)
- Test in dev environment before deploying
- Monitor import errors (first sign of mismatch)

### Last Updated
2026-04-11 (upgraded from 0.42b0)
```

---

## Upgrade Path: v0.x → v0.y (patch/minor within v0)

**Safe within v0 series if compatible:**

1. Update all packages to new minor version together:
   ```bash
   pip install \
     opentelemetry-api==0.45b0 \
     opentelemetry-sdk==0.45b0 \
     opentelemetry-exporter-otlp==0.45b0 \
     opentelemetry-instrumentation-requests==0.45b0
   ```

2. Test application locally
3. Deploy

---

## Upgrade Path: v0.x → v1.x (major, breaking)

**API and exporter change:**

1. **Read v0→v1 migration guide** (official OpenTelemetry docs)
2. **Update application code** to use v1 API
3. **Update requirements.txt** to v1 versions
4. **Update exporter:** v0 Jaeger → v1 OTLP
5. **Test locally** with new code and v1 dependencies
6. **Deploy**

---

## References

- [OpenTelemetry Official Docs](https://opentelemetry.io/docs/)
- [OpenTelemetry Python](https://github.com/open-telemetry/opentelemetry-python)
- [OpenTelemetry JavaScript](https://github.com/open-telemetry/opentelemetry-js)
- [OpenTelemetry Go](https://github.com/open-telemetry/opentelemetry-go)
- [OpenTelemetry Java](https://github.com/open-telemetry/opentelemetry-java)
- [Compatibility Matrix](https://opentelemetry.io/docs/reference/specification/protocol/)
