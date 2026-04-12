---
name: infra-version-management
description: Use when adding infrastructure services, upgrading versions, or troubleshooting version mismatch errors - prevents cascade failures from incompatible components and breaking changes
---

# Infrastructure Version Management

**Core principle:** Explicit version pinning, compatibility verification, and breaking change review prevent cascade failures. Major version upgrades are architectural rewrites, not incremental changes.

---

## When to Use

- Adding any infrastructure service or component
- Upgrading infrastructure component versions
- Integrating interdependent components (SDKs, exporters, libraries, etc.)
- Troubleshooting version-mismatch errors (import errors, config rejection, compatibility failures)
- Setting up docker-compose, Kubernetes, or other infrastructure configs

---

## Pre-Service Checklist: Before First Deploy

### 1. Pin Versions Explicitly

**Never use `latest` tags** — they can be stale or cached.

```yaml
# ❌ DON'T
my-service:
  image: registry.example.com/my-service:latest

# ✅ DO
my-service:
  image: registry.example.com/my-service:2.1.0
  # With comment: registry.example.com/my-service v2.1.0 (2024-12-01)
  # See: infrastructure-versions.md
```

**Action:** Create `infrastructure-versions.md` listing all services with explicit versions and release dates.

---

### 2. Verify Package/Image Name Matches Version

**Common pitfall:** Package or image names change between major versions.

```yaml
# Problem: old-package:latest is v1 (deprecated or removed)
service:
  image: registry.example.com/old-package:latest  # This is v1!

# Solution: v2 might use a different package/image name entirely
service:
  image: registry.example.com/new-package:2.0.0  # Different name
```

**Action:** Verify for each component:
- Does the package/image name match the version you want?
- Check official documentation for version → package name mapping
- Is `latest` actually current? (Pull and check)
- When was the package last updated?

---

### 3. Read Breaking Changes FIRST

**Before upgrading any major version:** Read the official migration guide.

```markdown
### Example: Service v2 → v3 Breaking Changes

1. Read official release notes for breaking changes (GitHub, docs, or changelog)
2. Note all required changes:
   - Removed config fields
   - Renamed or moved config sections
   - New required fields
   - Architectural changes
3. Update local config file before deploying
4. Test locally
5. Only then deploy
```

**Red flags:**
- Major version bump (v1→v2, v2→v3)
- Mention of "breaking changes", "migration required", or "config update"
- Architectural changes or new approach to core functionality

---

### 4. Verify Interdependent Component Compatibility

**Critical rule:** When components depend on each other (SDK + exporter, library + plugin, etc.), they must be compatible.

Mixing incompatible versions causes import errors, API mismatches, and cascading failures.

**Pattern:** Identify dependency relationships:
- Core + Extensions (SDK + Exporter, Library + Plugin)
- Client + Server (API + Backend)
- Framework + Instrumentation
- Any listed as "peer dependencies" or "required with"

**Action:**
1. Check if components have a compatibility matrix (GitHub, docs, changelog)
2. Pin dependent components to compatible versions
3. Document the dependency relationship in infrastructure-versions.md
4. Test together in dev before committing

**Generic example:**
```
# Dependency relationships to check:
- Component A (core) requires Component B (plugin) version 2.x
- Component B v3.x is incompatible with Component A v1.x
- Therefore: pin both Component A and B to versions that work together
```

---

### 5. Test Configuration Locally First

**Before deploying anywhere:** Validate and test your infrastructure config locally.

**Generic workflow:**
1. **Validate syntax** — Use your deployment tool's validator
2. **Start locally** — Spin up services in local environment
3. **Wait & verify health** — Give services time to initialize, check health (30 seconds typical)
4. **Check logs** — Review startup logs for errors
5. **Verify connectivity** — Confirm services can communicate
6. **Only after all checks pass** — Commit and push

**Deployment-specific examples:**

Docker Compose:
```bash
docker-compose config > /dev/null || exit 1  # Validate syntax
docker-compose up                             # Start services
sleep 30 && docker-compose ps                 # Check health
docker-compose logs | head -50                # Check startup logs
docker-compose down                           # Cleanup
```

Kubernetes:
```bash
kubectl apply -f manifests/ --dry-run=client  # Validate syntax
kubectl apply -f manifests/                   # Start services
sleep 30 && kubectl get pods                  # Check health
kubectl logs -f deployment/my-service         # Check startup logs
kubectl delete -f manifests/                  # Cleanup
```

Terraform:
```bash
terraform validate                            # Validate syntax
terraform plan                                # Preview changes
terraform apply                               # Start services
sleep 30 && terraform show                    # Check state
terraform destroy                             # Cleanup
```

---

## Troubleshooting: Common Version Mismatch Patterns

### Missing Module / Dependency Error

**Pattern:** Error mentions missing module or dependency

**Likely cause:** Dependent components have incompatible versions

**Fix:**
1. Check what each component requires (docs, compatibility matrix)
2. Verify all dependent components are compatible versions
3. Pin exact versions of all dependencies
4. Test together locally before deploying

---

### Unknown Flag / Config Syntax Error

**Pattern:** Service rejects a command-line flag or config field it used to accept

**Likely cause:** Major version upgrade changed config syntax or removed flags

**Fix:**
1. Read the breaking changes section for that major version
2. Check what the new config syntax should be
3. Update your config to new format
4. Test locally before deploying

---

### Runtime Connectivity Error ("Connection refused", "Timeout")

**Pattern:** Service starts but other services can't communicate with it

**Likely cause:** Config binds to localhost/127.0.0.1 only; needs 0.0.0.0 for Docker/network access

**Fix:**
1. Check binding configuration (what host/port is the service listening on?)
2. Ensure it binds to 0.0.0.0 or a network-accessible address, not localhost
3. Verify network connectivity between services
4. Test locally before deploying

---

## Deliverable: Infrastructure Versions Document

Create `infrastructure-versions.md` at project root:

```markdown
# Infrastructure Versions

Last updated: 2026-04-11

## Services

| Service | Version | Package/Image | Breaking Changes | Compatibility Notes | Status |
|---------|---------|---|-----|---|---|
| service-a | 2.1.0 | registry/service-a:2.1.0 | v1→v2 architectural change | None | ✅ Verified |
| service-b | 3.0.0 | registry/service-b:3.0.0 | v2→v3 config migration needed | Compatible with service-a v2.x | ✅ Verified |
| service-c | 1.5.0 | registry/service-c:1.5.0 | None in v1.x | Requires service-a ≥ v2.0 | ✅ Verified |

## Interdependent Components

Document any components that must be compatible:
- service-a + service-c: Both must be v2.x + v1.x (incompatible if mixed)
- service-b: Must match service-a minor version (e.g., both v2.x)

## Testing Checklist

- [ ] All versions pinned explicitly (no `latest` tags)
- [ ] Package/image name verified to match version
- [ ] Breaking changes documented for major versions
- [ ] Dependent components verified compatible
- [ ] Config syntax matches each service version
- [ ] All configs validate locally
- [ ] All services start and pass health check
- [ ] Services can communicate with each other
- [ ] No initialization errors in logs (first 30 seconds)
```

---

## Quick Reference: The Workflow

### 1. Before Adding Any Service

- [ ] Pin version explicitly (never `latest`)
- [ ] Verify package/image name matches version
- [ ] Read breaking changes for that version
- [ ] Check if dependent on other components (SDK, exporter, etc.)
- [ ] Document in infrastructure-versions.md

### 2. Before First Deployment

- [ ] Validate config syntax locally
- [ ] Test service startup locally
- [ ] Check health/startup logs
- [ ] Verify service responds to health checks
- [ ] Check connectivity with other services
- [ ] Only then deploy

### 3. When Upgrading

- [ ] Read breaking changes for major versions
- [ ] Update config for new syntax/requirements
- [ ] Pin new version explicitly
- [ ] Test locally first
- [ ] Verify health after upgrade
- [ ] Document version change

### 4. When Troubleshooting Version Errors

- [ ] Identify error pattern (missing module, config syntax, connectivity)
- [ ] Check if this is a known breaking change for that version
- [ ] Verify all dependent components are compatible versions
- [ ] Fix one issue at a time; test after each fix
- [ ] Check logs for clues

**Key principle:** Explicitness and verification up-front saves hours of debugging later.

---

## Library-Specific Guides

For detailed guidance on specific infrastructure components, see supplementary guides organized by category:

- **Metrics/** — Version management for metrics collection systems
- **Logging/** — Version management for log aggregation systems
- **Telemetry/Tracing/** — Version management for distributed tracing systems and instrumentation SDKs

Each guide extends the core principles with library-specific examples, breaking changes, and configuration patterns.

**Example:** If managing a metrics system, consult the Metrics category guide for version pinning examples, known breaking changes, and config patterns specific to that system.
