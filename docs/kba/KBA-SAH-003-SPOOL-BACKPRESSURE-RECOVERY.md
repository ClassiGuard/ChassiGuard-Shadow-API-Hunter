# KBA-SAH-003: Resolving Spool Backpressure & Disk Capacity Alerts

**Article ID:** KBA-SAH-003  
**Category:** Resilience & Storage  
**Severity:** Tier 2 (High)  
**Target Roles:** Platform Operations, Site Reliability Engineers (SRE), Linux Administrators  
**Applies To:** Shadow API Hunter Enterprise v1.0.0+ (`shadowapi-collector`, `shadowapi-spool`, `shadowapi-sync`)  

---

## 1. Problem Description & Operational Symptoms

The durable disk spool buffer under `/var/spool/shadowapi/` guarantees that high-velocity API traffic is captured safely without memory exhaustion or data loss. However, if downstream processing slows or the destination storage exceeds quotas:

1. **HTTP 429 Backpressure**: `shadowapi-collector` begins rejecting incoming API observation traffic from Envoy/NGINX/gateways with:
   ```json
   {"error":"Spool capacity exceeded; backpressure dropping events"}
   ```
2. **Prometheus Metrics**:
   - `observations_dropped_capacity_total > 0`
   - `spool_bytes_current >= spool_bytes_max`
3. **System Alert**: SRE monitoring receives alert `ShadowApiSpoolCapacityExceeded` (severity: HIGH).

---

## 2. Technical Architecture: Partitioned Spool Lifecycle

The spool directory uses strict partition isolation and AES-256-GCM envelope encryption:

```text
/var/spool/shadowapi/
Γö£ΓöÇΓöÇ queue/         <- Raw telemetry from collectors (AES-256-GCM sealed)
Γö£ΓöÇΓöÇ normalized/    <- Privacy-scrubbed & path-folded records from Rust ingest
Γö£ΓöÇΓöÇ outbox/        <- Events awaiting dispatch to inventory/risk engines
Γö£ΓöÇΓöÇ archive/       <- Dispatched records preserved for audit (retention-managed)
ΓööΓöÇΓöÇ deadletter/    <- Malformed or corrupt records quarantined for inspection
```

Each segment file adheres to the **`SAHS` v1 binary framing** specification with 12-byte random nonces, 32-bit IEEE CRC32 checksums, and mandatory `fsync` barriers.

---

## 3. Step-by-Step Recovery Procedure

### Step 1: Assess Current Spool Disk Usage & Quota State

Log into the affected Linux node and inspect the storage partition sizes:

```bash
# Check filesystem space
df -h /var/spool/shadowapi

# Check spool partition byte usage
du -sh /var/spool/shadowapi/*

# Query live spool metrics from the collector
curl -s http://127.0.0.1:9101/metrics | grep -E "spool_bytes|observations_dropped"
```

---

### Step 2: Trigger Outbox Flush & Replay Delivery

If downstream services (e.g. `shadowapi-inventory` or SIEM/evidence destinations) were temporarily offline, force an immediate outbox dispatch replay:

```bash
# 1. Ensure inventory reconciler is running and healthy
systemctl status shadowapi-inventory

# 2. Trigger on-demand spool flush
curl -X POST http://127.0.0.1:9103/api/v1/spool/sync
```

Watch the queue drain in real time:
```bash
watch -n 1 "curl -s http://127.0.0.1:9101/metrics | grep spool_bytes_current"
```

---

### Step 3: Archive Retention Cleanup

If `archive/` has accumulated historical records exceeding the compliance retention lease:

```bash
# Safely purge archived segments older than 7 days
find /var/spool/shadowapi/archive/ -type f -name "*.spool" -mtime +7 -delete

# Confirm recovered disk space
df -h /var/spool/shadowapi
```

> [!CAUTION]
> Never manually delete active files from `queue/` or `outbox/`. Deleting active segments will cause event sequence gaps and corrupt partition indexes. Always use the built-in purge API.

---

### Step 4: Dynamically Increase Spool Capacity Quota

If baseline telemetry volume permanently exceeds the configured maximum quota (default: 10 GB):

1. Edit the collector and spool service configuration in `/etc/shadowapi/collector.conf`:
```ini
# Increase quota to 25 GB (25600 MB)
MAX_SPOOL_MB=25600
```

2. Reload and restart the services gracefully:
```bash
sudo systemctl reload-or-restart shadowapi-collector shadowapi-spool
```

3. Confirm new limit in metrics:
```bash
curl -s http://127.0.0.1:9101/metrics | grep spool_bytes_max
# Expected: "spool_bytes_max": 26843545600
```

---

### Step 5: Torn-Write Recovery (Post Power-Failure / Crash)

If the server suffered an unexpected kernel panic or power outage, corrupt partial segments may prevent queue rotation.

Execute the built-in torn-write recovery tool:
```bash
curl -X POST http://127.0.0.1:9103/api/v1/spool/recover
```
Expected output:
```json
{
  "status": "recovered",
  "valid_records_retained": 48210,
  "torn_writes_truncated": 1,
  "quarantined_to_deadletter": 0
}
```

---

## 4. Health Verification & Recovery Confirmation

1. Check Readiness Probe:
```bash
curl -s http://127.0.0.1:9101/health/ready
# Must return: {"status":"ready","spool_queue":"writable"}
```
2. Verify backpressure error has cleared on edge gateways (Envoy/NGINX logs should return `202 Accepted`).
3. Confirm `observations_dropped_capacity_total` has ceased incrementing.
