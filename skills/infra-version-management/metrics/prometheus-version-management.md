# Prometheus Version Management

**Supplement to:** [infra-version-management SKILL.md](SKILL.md)

Use this guide when managing Prometheus versions and upgrades. Prometheus is generally stable within major versions, but v1→v2 has breaking changes.

---

## Quick Facts

| Version | Status | Key Changes | Image |
|---------|--------|-------------|-------|
| 1.x | Legacy | - | `prom/prometheus:1.x` |
| 2.x | Current stable | Minimal breaking changes within 2.x | `prom/prometheus:v2.x` |
| 3.x (future) | TBD | Monitor releases | TBD |

---

## Version Pinning Example

```yaml
# docker-compose.yml
prometheus:
  image: prom/prometheus:v2.45.0  # Pin explicitly, not latest
  volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml
    - prometheus_data:/prometheus
  ports:
    - "9090:9090"

volumes:
  prometheus_data:
```

---

## Breaking Changes: v1 → v2

**Major changes when upgrading from v1 to v2:**

1. **Command-line flags** — Some flags renamed or removed
   - v1: `--storage.local.path=/prometheus`
   - v2: `--storage.tsdb.path=/prometheus`

2. **Config format** — Remains YAML but some fields changed
   - Service discovery configuration refactored
   - Scrape config options updated

3. **Storage format** — TSDB format completely different
   - v1 and v2 cannot share the same data directory
   - Migration: back up v1 data, start fresh with v2, or use retention to expire old data

4. **API changes** — HTTP API endpoints remain compatible for most operations

---

## Configuration Example: Single-Instance v2

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'prometheus'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'app'
    static_configs:
      - targets: ['localhost:8080']
```

---

## Testing Before Deployment

```bash
# Validate config syntax
docker run --rm -v $(pwd):/etc/prometheus prom/prometheus:v2.45.0 \
  --config.file=/etc/prometheus/prometheus.yml \
  --dry-run

# Or with docker-compose
docker-compose config

# Start locally
docker-compose up prometheus

# Check health
curl http://localhost:9090/-/healthy

# Verify metrics are collected (wait 30 seconds)
sleep 30
curl http://localhost:9090/api/v1/query?query=up
```

---

## Upgrade Path: v2.x → v2.y (patch/minor)

**Safe to upgrade within same major version (e.g., v2.40 → v2.45):**

1. Back up prometheus data: `cp -r /prometheus /prometheus.backup`
2. Update image tag in docker-compose.yml
3. Restart: `docker-compose down && docker-compose up`
4. Verify health: `curl http://localhost:9090/-/healthy`
5. Check metrics: `curl http://localhost:9090/api/v1/targets`
6. If issues, restore: `docker-compose down && rm -rf /prometheus && mv /prometheus.backup /prometheus`

---

## Upgrade Path: v1.x → v2.x (major, breaking)

**Data migration required:**

1. **Back up v1 data:** `cp -r /prometheus /prometheus.v1.backup`
2. **Decide:**
   - **Option A (Keep data):** Complex migration, rare for Prometheus since metrics expire anyway
   - **Option B (Fresh start):** Remove old data, start fresh with v2 (recommended for MVP)
3. **Update config** to v2 syntax if you have custom scrape configs
4. **Test locally** with v2 image and empty TSDB
5. **Deploy v2** with fresh data directory
6. **Verify:** Metrics start collecting immediately

---

## Common Issues

### "unknown flag --storage.local.path"

**Cause:** Using v1 command-line flags with v2

**Fix:** Use v2 flag: `--storage.tsdb.path=/prometheus`

---

### "error loading config: yaml: line X: field 'xxx' not found"

**Cause:** Config syntax changed between v1 and v2

**Fix:** Check Prometheus v2 config examples and update YAML structure

---

### Metrics disappear after upgrade

**Cause:** v2 uses different TSDB format; old v1 data incompatible

**Fix:** This is expected. v2 starts fresh and begins collecting immediately. Retention will drop old v1 metrics over time.

---

## monitoring infrastructure-versions.md

```markdown
## Prometheus

- Version: v2.45.0
- Image: prom/prometheus:v2.45.0
- Release Date: 2024-08-10
- Breaking Changes: None (within v2.x)
- Data Format: v2 TSDB (incompatible with v1)
- Health Check: GET /-/healthy
- Status: ✅ Verified

### Upgrade Notes
- Within v2.x: Safe to upgrade (back up data, test locally)
- v1 → v2: Requires fresh TSDB, config migration
- Last upgraded: 2026-04-11 (from v2.40 → v2.45)
```

---

## References

- [Official Prometheus Docs](https://prometheus.io/docs/)
- [Prometheus Docker Image](https://hub.docker.com/r/prom/prometheus/)
- [v1 to v2 Migration Guide](https://prometheus.io/docs/prometheus/latest/migration/)
