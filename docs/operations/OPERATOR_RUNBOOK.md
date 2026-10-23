# Shadow API Hunter Enterprise ΓÇö SOC & AppSec Operator Runbook

**Document ID:** RUNBOOK-OPS-001  
**Target Roles:** SOC Analysts (Tier 1/2/3), AppSec Incident Responders, On-Call Security Engineers  
**Applies To:** Shadow API Hunter Enterprise v1.0.0+  

---

## 1. Alert Catalog & Severity Matrix

The deterministic risk evaluation engine ([`shadowapi-risk`](file:///d:/Uday%20Projects/Shadow-API-Hunter-Enterprise/services/linux/risk/)) continuously scans reconciled API observations against formal threat rules.

| Alert Rule ID | Description | Default Risk Score | Severity | Response SLA | Primary Action |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`RULE-SHADOW-001`** | Live API route observed without entry in approved OpenAPI spec or CMDB | 75.0 - 90.0 | **HIGH** | 8 Hours | Triage via [KBA-SAH-004](file:///d:/Uday%20Projects/Shadow-API-Hunter-Enterprise/docs/kba/KBA-SAH-004-SHADOW-API-TRIAGE-REMEDIATION.md); assign owner or block |
| **`RULE-ZOMBIE-001`** | Active production calls against deprecated / retired endpoint | 85.0 - 95.0 | **CRITICAL** | 2 Hours | Triage via [KBA-SAH-005](file:///d:/Uday%20Projects/Shadow-API-Hunter-Enterprise/docs/kba/KBA-SAH-005-ZOMBIE-API-DEPRECATION.md); notify callers and sunset |
| **`RULE-DATA-001`** | Unmanaged or unauthenticated route transmitting sensitive data classes | 80.0 - 100.0 | **CRITICAL** | 2 Hours | Audit card/PII exposure via [KBA-SAH-006](file:///d:/Uday%20Projects/Shadow-API-Hunter-Enterprise/docs/kba/KBA-SAH-006-ZERO-LEAKAGE-AUDIT.md) |
| **`RULE-AUTH-001`** | Missing auth headers, expired JWTs, or rapid credential elevation | 70.0 - 85.0 | **HIGH** | 8 Hours | Inspect AuthSignal telemetry; verify OAuth2 gateway policies |
| **`RULE-BOLA-001`** | Single client accessing excessive horizontal object resource IDs | 75.0 - 85.0 | **HIGH** | 8 Hours | Evaluate BOLA hypothesis; schedule scoped verification test |
| **`RULE-ROGUE-001`** | Discovered API host domain outside enterprise DNS zones | 90.0 - 100.0 | **CRITICAL** | 2 Hours | Verify IP origin; block at perimeter WAF if rogue server |

---

## 2. Infrastructure & Fleet Health Alarms

| Alarm Name | Trigger Condition | Severity | Playbook Reference | Immediate Action |
| :--- | :--- | :--- | :--- | :--- |
| `ShadowApiSpoolCapacityExceeded` | Collector queue disk usage > 90% | **HIGH** | [KBA-SAH-003](file:///d:/Uday%20Projects/Shadow-API-Hunter-Enterprise/docs/kba/KBA-SAH-003-SPOOL-BACKPRESSURE-RECOVERY.md) | Flush outbox via `/api/v1/spool/sync` or expand quota |
| `ShadowApiPolicyAutonomyExpired` | Policy lease remaining = 0s | **CRITICAL** | [KBA-SAH-002](file:///d:/Uday%20Projects/Shadow-API-Hunter-Enterprise/docs/kba/KBA-SAH-002-POLICY-AUTONOMY-EXPIRED.md) | Re-sign and push snapshot from Windows Control Plane |
| `ShadowApiKillSwitchEngaged` | Active verifier emergency halt active | **HIGH** | [KBA-SAH-007](file:///d:/Uday%20Projects/Shadow-API-Hunter-Enterprise/docs/kba/KBA-SAH-007-ACTIVE-VERIFICATION-KILLSWITCH.md) | Investigate target error rate; reset when safe |
| `ShadowApiDatabaseDisconnected` | Management API SQL connection lost | **CRITICAL** | [KBA-SAH-010](file:///d:/Uday%20Projects/Shadow-API-Hunter-Enterprise/docs/kba/KBA-SAH-010-SQL-SERVER-FAILOVER-DISASTER-RECOVERY.md) | Trigger Always On failover or restart SQL services |

---

## 3. Tier 1 / Tier 2 Triage Playbook

### Step 1: Ingest Alarm Receipt in SIEM
Alarms stream to Splunk / Microsoft Sentinel / Elastic under index `shadowapi-events`:
```spl
index=shadowapi-events rule_id="RULE-SHADOW-001" | table _time, event_id, normalized_route, risk_score, sensitive_classes
```

### Step 2: Corroborate in Management UI
1. Log into `https://shadowapi.corp.internal/`.
2. Locate the incident by `event_id` or `normalized_route`.
3. Check the **Telemetry Signal Box**:
   - Has this route been called before? (Check First Seen / Frequency).
   - What data classes were detected? (`CREDIT_CARD`, `EMAIL`, `PHONE`, etc.).
   - Is authentication present? (`AuthPresent = true/false`, `AuthScheme`).

### Step 3: Dispatch Incident Ticket
If the finding is legitimate and requires developer intervention:
- Click **Dispatch Ticket** in the UI.
- Choose destination: **Jira** (Development Bug) or **ServiceNow** (Security Incident).
- Select Priority based on SLA table above.

---

## 4. Operational Cadences & Maintenance Schedule

### Daily Cadence (SOC Morning Checklist)
1. Verify **Fleet Health Overview** (confirm 100% of nodes report `HEALTHY`).
2. Triage all new **Critical/High findings** discovered in the last 24 hours.
3. Check **Spool Partition Depths** across all sensor nodes (confirm queue < 20% capacity).

### Weekly Cadence (AppSec Review)
1. Import updated **OpenAPI Specifications** from CI/CD repositories into Tab 2.
2. Review **Spec Diff & Drift** (Tab 5) to identify newly introduced breaking changes.
3. Trigger **CMDB Bi-directional Sync** to reconcile asset ownership changes.

### Monthly Cadence (Compliance Audit)
1. Export **Compliance & Audit Package** from Tab 6.
2. Verify cryptographic HMAC signature and Merkle root against evidence archive vault.
3. Review suppressed findings leases and expire outdated suppressions.
