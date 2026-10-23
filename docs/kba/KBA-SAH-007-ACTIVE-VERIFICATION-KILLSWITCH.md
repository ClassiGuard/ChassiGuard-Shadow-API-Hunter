# KBA-SAH-007: Active Verification Emergency Kill Switch & Anti-SSRF Violations

**Article ID:** KBA-SAH-007  
**Category:** Verification Safety & Guardrails  
**Severity:** Tier 1 (Critical Safety Control)  
**Target Roles:** SOC Operators, AppSec Engineers, Site Reliability Engineers (SRE)  
**Applies To:** Shadow API Hunter Enterprise v1.0.0+ (`shadowapi-verifier` daemon)  

---

## 1. Operational Overview & Safety Mandate

Active verification in Shadow API Hunter operates under strict governance (**ADR-0005** and **ADR-0007**):
- **Passive Discovery by Default**: Automated network fuzzing or intrusive probing is strictly prohibited.
- **Explicit Operator Scope**: Active tests execute only against registered target scopes with human-approved test DSL suites.
- **Safety Engine & Anti-SSRF**: The built-in network dialer blocks cloud metadata endpoints (`169.254.169.254`, `metadata.google.internal`), loopback addresses, link-local IPs, and un-scoped RFC 1918 private subnets with socket-level anti-DNS rebinding protection.
- **Emergency Kill Switch**: Any operator or automated guardrail can instantly halt all running and scheduled active verifications across the cluster.

---

## 2. Emergency Kill Switch Activation

If target application performance degrades, rate limits are tripped, or unexpected 5xx errors occur during an active verification run:

### Method A: Instant Activation via Management UI

1. On any screen in the IIS Management UI, locate the **Active Verifier Status Pill** in the top navigation bar.
2. Click the red **Emergency Kill Switch** button.
3. Confirm the modal prompt: `Trigger Global Verifier Halt`.

---

### Method B: Immediate CLI / cURL Execution (Direct Node)

Log into the verifier host and trigger the kill switch via the local API:

```bash
curl -X POST http://127.0.0.1:9110/api/v1/verifier/kill-switch \
    -H "Content-Type: application/json" \
    -d '{
      "operator_id": "operator-secops-01",
      "reason": "Target gateway error rate exceeded 5% threshold during testing"
    }'
```

Expected HTTP Response (`200 OK`):
```json
{
  "status": "halted",
  "kill_switch_active": true,
  "in_flight_cancelled": 4,
  "timestamp": "2026-10-22T11:22:31Z"
}
```

### System Behavior During Kill Switch State:
- All active in-flight HTTP probe contexts are immediately cancelled (`context.Canceled`).
- Token bucket rate-limiter workers halt queue processing.
- Any subsequent verification request returns `HTTP 409 Conflict`:
  ```json
  {"error":"Active verification halted: global kill switch is engaged"}
  ```

---

## 3. Investigating Anti-SSRF Violations

If a verification test fails with an anti-SSRF error:

```text
[verifier] Prohibited target IP 169.254.169.254: SSRF tripwire triggered
```

1. Inspect the verifier safety logs:
```bash
journalctl -u shadowapi-verifier -n 100 --no-pager | grep -E "SSRF|Blocked|Tripwire"
```
2. Root Causes:
   - **Cloud Metadata Probe**: The test fixture attempted to resolve or connect to `169.254.169.254` (AWS/GCP metadata service). This is permanently blocked by the kernel-level socket dialer.
   - **DNS Rebinding**: Target domain resolved to an internal IP address outside the approved scope subnet.

---

## 4. Reset & Resumption Procedure

Once target systems are verified healthy and the issue is resolved:

1. Validate target system status and ensure no residual traffic pressure.
2. Reset the kill switch via authorized administrative credentials:

```bash
curl -X POST http://127.0.0.1:9110/api/v1/verifier/kill-switch/reset \
    -H "Content-Type: application/json" \
    -d '{
      "operator_id": "lead-architect@corp.internal",
      "justification": "Root cause identified as temporary database lock; capacity verified"
    }'
```

3. Confirm verifier readiness probe:
```bash
curl -s http://127.0.0.1:9110/health/ready
# Expected: {"status":"ready","kill_switch":"inactive","active_scopes":2}
```
