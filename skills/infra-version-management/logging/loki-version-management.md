# Loki Version Management

**Supplement to:** [infra-version-management SKILL.md](SKILL.md)

Use this guide when managing Grafana Loki versions and upgrades. Loki has breaking changes in major version transitions (v2→v3).

---

## Quick Facts

| Version | Status | Storage | Notable Changes |
|---------|--------|---------|-----------------|
| 2.x | Current stable | Multiple options | BoltDB, S3, GCS |
| 3.x | Latest | Migrated to TSDB | v2→v3 config migration required |

---

## Version Pinning Example

```yaml
# docker-compose.yml
loki:
  image: grafana/loki:3.0.0  # Pin explicitly, not latest
  volumes:
    - ./loki-config.yaml:/etc/loki/local-config.yaml
    - loki_data:/loki
  ports:
    - "3100:3100"
  command: -config.file=/etc/loki/local-config.yaml

volumes:
  loki_data:
```

---

## Breaking Changes: v2 → v3

**Major config changes when upgrading from v2 to v3:**

### Removed Fields (v2 → v3)

```yaml
# ❌ These fields removed in v3
ingester:
  enforce_metric_name: true  # REMOVED
  retention_deletes_enabled: true  # REMOVED
  max_chunk_age: 2h  # Changed

distributor:
  rate_limit_enabled: true  # REMOVED
```

### Renamed/Moved Fields

```yaml
# ❌ v2
schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper

# ✅ v3
schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
```

### Storage Format Changes

- **v2:** BoltDB shipper with index
- **v3:** Native TSDB (new format, incompatible with v2)

---

## Configuration Example: Single-Instance v3

```yaml
# loki-config.yaml - v3 MVP single-instance
auth_enabled: false

ingester:
  chunk_idle_period: 3m
  chunk_retain_period: 1m
  lifecycler:
    ring:
      kvstore:
        store: inmemory  # Single-instance: use inmemory
      replication_factor: 1

schema_config:
  configs:
    - from: 2023-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  filesystem:
    directory: /loki/chunks
  index_cache_validity: 5m

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

query_scheduler:
  max_cache_freshness_per_query: 10m

server:
  http_listen_port: 3100
  log_level: info
```

---

## Docker Compose Example

```yaml
version: '3'
services:
  loki:
    image: grafana/loki:3.0.0
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki_data:/loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml

  promtail:  # Log shipper
    image: grafana/promtail:3.0.0
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail-config.yaml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    depends_on:
      - loki

volumes:
  loki_data:
```

---

## Promtail (Log Shipper) Configuration

```yaml
# promtail-config.yaml - v3 compatible
clients:
  - url: http://loki:3100/loki/api/v1/push

positions:
  filename: /tmp/positions.yaml

scrape_configs:
  - job_name: docker
    docker: {}
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        target_label: 'job'
      - source_labels: ['__meta_docker_container_id']
        target_label: 'container_id'
```

---

## Testing Before Deployment

```bash
# Validate config syntax
docker run --rm -v $(pwd):/etc/loki grafana/loki:3.0.0 \
  -config.file=/etc/loki/local-config.yaml \
  -log.level=debug

# Start locally
docker-compose up loki

# Wait for startup
sleep 10

# Check health
curl http://localhost:3100/ready

# Check API
curl http://localhost:3100/loki/api/v1/labels

# Push test logs
curl -X POST http://localhost:3100/loki/api/v1/push \
  -H "Content-Type: application/json" \
  -d '{
    "streams": [
      {
        "stream": {"job": "test"},
        "values": [["'$(date +%s)'000000000", "test message"]]
      }
    ]
  }'

# Query logs
curl 'http://localhost:3100/loki/api/v1/query_range?query={job="test"}'
```

---

## Upgrade Path: v2.x → v3.x (major, breaking)

**Data migration required — plan carefully:**

1. **Back up v2 data:** `cp -r /loki /loki.v2.backup`
2. **Read v2→v3 migration guide** (you're doing this now ✓)
3. **Update config file** from v2 to v3 format
4. **Test v3 locally** with empty data directory (fresh start)
5. **Deploy v3** alongside v2 if needed (separate data directory)
6. **Migrate logs:**
   - **Option A:** Fresh start (simple, recommended for MVP)
   - **Option B:** Export v2 → import to v3 (complex, for production)
7. **Update Promtail** to v3 (should be compatible)
8. **Verify queries work** in v3 with new data

---

## Common Issues

### "error loading config: yaml: line X: field 'xxx' not found"

**Cause:** Using v2 config fields in v3 (they were removed)

**Fix:** Remove v2-only fields; see Breaking Changes section above

---

### Loki starts but logs don't appear

**Cause:** Promtail still targeting v2 endpoints; schema mismatch

**Fix:**
1. Update Promtail to v3
2. Verify Loki config has correct schema version
3. Check Promtail can reach Loki: `curl http://loki:3100/ready`

---

### "too many validation errors in config"

**Cause:** Mixing v2 and v3 config fields

**Fix:** Start with v3 single-instance example above; add fields one at a time

---

## Monitoring infrastructure-versions.md

```markdown
## Loki

- Version: 3.0.0
- Image: grafana/loki:3.0.0
- Release Date: 2024-10-15
- Breaking Changes: v2→v3 config migration; storage format change (TSDB); removed fields
- Storage: /loki (mounted volume)
- API Port: 3100
- Health Check: GET /ready
- Status: ✅ Verified

### Dependencies
- Promtail: 3.0.0 (log shipper, must match Loki version)

### Upgrade Notes
- v2.x: BoltDB shipper, different config structure
- v3.x: Native TSDB, simplified config
- Data Migration: Not backward compatible; plan fresh start for MVP
- Promtail: Update to same version as Loki (3.0.0)
- Last upgraded: 2026-04-11 (v2.8.0 → v3.0.0)

### Known Issues
- Loki v3 disables WAL by default (OK for single-instance)
- Storage paths must use mounted volumes (not /var)
```

---

## Single-Instance vs Distributed

### Single-Instance (MVP) - Use This

```yaml
ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory  # No external service needed
      replication_factor: 1  # Single instance

storage_config:
  filesystem:  # Local disk
    directory: /loki/chunks

schema_config:
  configs:
    - store: tsdb
      object_store: filesystem
```

### Distributed (Production) - Different Setup

```yaml
ingester:
  lifecycler:
    ring:
      kvstore:
        store: consul  # External discovery
      replication_factor: 3  # HA

storage_config:
  s3:  # Distributed storage
    bucket_name: my-loki-bucket

schema_config:
  configs:
    - store: tsdb
      object_store: s3  # Remote storage
```

---

## References

- [Loki Official Docs](https://grafana.com/docs/loki/latest/)
- [Loki Docker Images](https://hub.docker.com/r/grafana/loki)
- [Loki v2 to v3 Migration](https://grafana.com/docs/loki/latest/release-notes/v3-breaking-changes/)
- [Promtail Configuration](https://grafana.com/docs/loki/latest/send-data/promtail/)
