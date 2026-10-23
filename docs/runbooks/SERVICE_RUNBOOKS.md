# Service Operational Runbooks & Triage Procedures
**Shadow API Hunter Enterprise** ΓÇö Operational Runbook Catalog

This runbook catalog provides system administrators, SecOps engineers, and platform operators with concrete diagnostic commands, log locations, alerting thresholds, and recovery workflows for all platform services.

---

## 1. Quick Reference: Service Port & Log Registry

| Service | OS | Port | Log File / Journal | Unit Name |
|---|---|---|---|---|
| Management UI | Windows | 80/443 | `C:\inetpub\logs\LogFiles\` | IIS Site `ShadowAPI-Web` |
| Management API | Windows | 8080 | `C:\ProgramData\ShadowAPI\logs\management-api.log` | `ShadowAPI.Management.Api` |
| Management Sync | Windows | - | `C:\ProgramData\ShadowAPI\logs\sync.log` | `ShadowAPI.Management.Sync` |
| Management Jobs | Windows | - | `C:\ProgramData\ShadowAPI\logs\jobs.log` | `ShadowAPI.Management.Jobs` |
| Collector | Linux | 9101 | `/var/log/shadowapi/collector.log` | `shadowapi-collector.service` |
| Ingest Normalizer | Linux | 9102 | `/var/log/shadowapi/ingest.log` | `shadowapi-ingest.service` |
| Durable Spool | Linux | 9103 | `/var/log/shadowapi/spool.log` | `shadowapi-spool.service` |
| Session Correlation | Linux | 9104 | `/var/log/shadowapi/session.log` | `shadowapi-session.service` |
| Inventory Reconciler | Linux | 9105 | `/var/log/shadowapi/inventory.log` | `shadowapi-inventory.service` |
| Signed Policy Cache | Linux | 9108 | `/var/log/shadowapi/policy.log` | `shadowapi-policy.service` |
| Linux Sync Worker | Linux | 9107 | `/var/log/shadowapi/sync.log` | `shadowapi-sync.service` |
| Risk Evaluation | Linux | 9106 | `/var/log/shadowapi/risk.log` | `shadowapi-risk.service` |
| AI Model Router | Linux | 9109 | `/var/log/shadowapi/ai.log` | `shadowapi-ai.service` |
| Active Verifier | Linux | 9110 | `/var/log/shadowapi/verifier.log` | `shadowapi-verifier.service` |
| Evidence Vault | Linux | 9111 | `/var/log/shadowapi/evidence.log` | `shadowapi-evidence.service` |
| Fleet Health Telemetry | Linux | 9112 | `/var/log/shadowapi/health.log` | `shadowapi-health.service` |

---

## 2. Windows Services Operational Runbooks

### Runbook WIN-001: ShadowAPI.Management.Api
- **Purpose:** Primary REST API for route inventory, risk findings, policy distribution, and user authentication.
- **Health Check:** `curl -s http://127.0.0.1:8080/health/ready` (Returns HTTP 200 `{"status":"ready"}`)
- **Diagnostics:**
  ```powershell
  # Check service status
  Get-Service -Name "ShadowAPI.Management.Api"
  # Inspect recent log entries
  Get-Content "C:\ProgramData\ShadowAPI\logs\management-api.log" -Tail 50
  # Verify port binding
  Get-NetTCPConnection -LocalPort 8080 -State Listen
  ```
- **Recovery Procedure:**
  ```powershell
  Restart-Service -Name "ShadowAPI.Management.Api" -Force
  ```
- **Triage Checklist:**
  1. If `/health/ready` returns HTTP 503: Verify Microsoft SQL Server connectivity (`sqlcmd -S localhost -d ShadowApiHunter -E -Q "SELECT 1"`).
  2. If port 8080 is in use by another process: Adjust port in `C:\ProgramData\ShadowAPI\config\management-api.json`.

---

## 3. Linux Data Plane Services Operational Runbooks

### Runbook LNX-001: shadowapi-spool (Durable Queue & Power-Loss Recovery)
- **Purpose:** Durable AES-256-GCM encrypted ring buffer with CRC32 binary framing and torn-write auto-recovery.
- **Health Check:** `curl -s http://127.0.0.1:9103/health/live`
- **Diagnostics:**
  ```bash
  systemctl status shadowapi-spool.service
  journalctl -u shadowapi-spool.service -n 50 --no-pager
  ls -lh /var/spool/shadowapi/queue/ /var/spool/shadowapi/deadletter/
  ```
- **Recovery Procedure:**
  ```bash
  # Step 1: Attempt standard restart
  sudo systemctl restart shadowapi-spool.service
  
  # Step 2: If corrupt segment detected, run offline recovery tool
  sudo /opt/shadowapi/bin/shadowapi-spool --recover /var/spool/shadowapi
  sudo systemctl restart shadowapi-spool.service
  ```

---

### Runbook LNX-002: shadowapi-policy (Signed Offline Autonomy Engine)
- **Purpose:** Validates Ed25519-signed policy snapshots and guarantees 168 hours of offline policy continuity.
- **Health Check:** `curl -s http://127.0.0.1:9108/health/ready`
- **Diagnostics:**
  ```bash
  systemctl status shadowapi-policy.service
  curl -s http://127.0.0.1:9108/api/v1/policy/status | jq .
  cat /var/lib/shadowapi/policy.cache | jq .snapshot.version_tag
  ```
- **Recovery Procedure:**
  ```bash
  # Step 1: Verify disk cache integrity
  ls -l /var/lib/shadowapi/policy.cache
  
  # Step 2: Restart service
  sudo systemctl restart shadowapi-policy.service
  
  # Step 3: If cache missing, ingest emergency signed snapshot via CLI
  sudo /opt/shadowapi/bin/shadowapi-policy --sign-snapshot /etc/shadowapi/emergency-policy.json
  ```

---

### Runbook LNX-003: shadowapi-ai (Model Gateway & Redaction Broker)
- **Purpose:** Pre-inference zero-leakage scrubbing, local Ollama model routing, and deterministic fallback.
- **Health Check:** `curl -s http://127.0.0.1:9109/health/live`
- **Diagnostics:**
  ```bash
  systemctl status shadowapi-ai.service
  curl -s http://127.0.0.1:9109/api/v1/ai/providers | jq .
  # Check local Ollama daemon
  curl -s http://127.0.0.1:11434/api/tags
  ```
- **Recovery Procedure:**
  ```bash
  # Even if Ollama is down, shadowapi-ai automatically falls back to deterministic rules
  # To restart Ollama:
  sudo systemctl restart ollama.service
  sudo systemctl restart shadowapi-ai.service
  ```

---

### Runbook LNX-004: shadowapi-verifier (Active Verification & Emergency Kill Switch)
- **Purpose:** Executes signed declarative synthetic probes with SSRF protection and emergency kill switch.
- **Health Check:** `curl -s http://127.0.0.1:9110/health/ready`
- **Emergency Kill Switch Invocation:**
  ```bash
  # Global emergency fleet kill switch
  curl -X POST -H "Content-Type: application/json" \
    -d '{"reason": "Immediate security halt", "triggered_by": "secops-operator"}' \
    http://127.0.0.1:9110/api/v1/verifier/kill-switch
    
  # Check kill switch state
  curl -s http://127.0.0.1:9110/api/v1/verifier/status | jq .
  
  # Reset kill switch after incident resolution
  curl -X POST http://127.0.0.1:9110/api/v1/verifier/kill-switch/reset
  ```

---

### Runbook LNX-005: shadowapi-health (Fleet Health & Telemetry Daemon)
- **Purpose:** Aggregates health across all local services, tracks spool depth, and monitors policy autonomy countdowns.
- **Health Check:** `curl -s http://127.0.0.1:9112/health/ready`
- **Diagnostics:**
  ```bash
  systemctl status shadowapi-health.service
  curl -s http://127.0.0.1:9112/api/v1/health/node | jq .
  curl -s http://127.0.0.1:9112/metrics | grep shadowapi_health
  ```
- **Recovery Procedure:**
  ```bash
  sudo systemctl restart shadowapi-health.service
  ```
