# KBA-SAH-009: Bi-directional CMDB & Backstage Catalog Sync Reconciliation

**Article ID:** KBA-SAH-009  
**Category:** Governance & Software Inventory  
**Severity:** Tier 2 (Low to Medium)  
**Target Roles:** Enterprise Architects, CMDB Administrators, AppSec Governance Leads  
**Applies To:** Shadow API Hunter Enterprise v1.0.0+ (CMDB Sync Engine)  

---

## 1. Operational Objective

In large enterprises, APIs cannot be managed effectively without knowing who owns them and what business services they support. The **CMDB Sync Adapter** bridges Shadow API Hunter with enterprise Configuration Management Databases:
1. **ServiceNow CMDB**: Ingests Configuration Items (CIs) from `cmdb_ci_appl` and `cmdb_ci_service`.
2. **Backstage Software Catalog**: Ingests declared `Component` and `API` entities from `catalog-info.yaml` files.
3. **Automated Attribution**: Automatically attributes verified team owners and service IDs to discovered unassigned APIs.
4. **Shadow Route Export**: Exports newly discovered, unapproved shadow endpoints to ServiceNow as unmanaged CIs for enterprise configuration tracking.

---

## 2. Auto-Attribution Matching Precedence

When reconciling observed routes against imported CIs, the engine evaluates matches in strict specificity order:
1. **Exact Route Prefix Match**: If a CI specifies `RoutePrefixes: ["/api/v1/orders"]`, any discovered route under `/api/v1/orders/...` is attributed to that CI's owner.
2. **Service Identifier Match**: If the observed service header matches a known CI `ServiceId`.
3. **Host Domain Match**: Matches based on host domain (e.g. `billing.corp.internal`).

> [!NOTE]
> **Preservation Invariant**: Auto-attribution only updates an asset if its current owner is missing, empty, or set to `"UNKNOWN"` / `"UNASSIGNED"`. Manually assigned human ownership is never overwritten.

---

## 3. Step-by-Step Operator Procedure

### Step 1: Importing Catalog Entities into Shadow API Hunter

**Importing Backstage Catalog Entities (YAML):**
```powershell
$backstageYaml = @"
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: orders-service
  description: Core Order Fulfillment Microservice
spec:
  type: service
  lifecycle: production
  owner: orders-team@corp.internal
  route_prefix: /api/v1/orders
"@

Invoke-RestMethod -Uri "https://shadowapi.corp.internal/api/v1/cmdb/ci/import" `
    -Method POST `
    -Headers @{ "Authorization" = "Bearer $AdminToken"; "Content-Type" = "application/yaml" } `
    -Body $backstageYaml
```

**Importing ServiceNow CMDB Table JSON:**
```powershell
$serviceNowJson = @{
    "result" = @(
        @{
            "sys_id"           = "ci-warehouse-9941"
            "name"             = "warehouse-inventory-service"
            "owned_by"         = "logistics-eng@corp.internal"
            "operational_status" = "1" # Operational
            "u_route_prefix"   = "/api/v2/warehouse"
        }
    )
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri "https://shadowapi.corp.internal/api/v1/cmdb/ci/import" `
    -Method POST `
    -Headers @{ "Authorization" = "Bearer $AdminToken"; "Content-Type" = "application/json" } `
    -Body $serviceNowJson
```

---

### Step 2: Triggering On-Demand Bidirectional Reconciliation

**Via UI:**
1. In the IIS Management UI top navigation, locate the **CMDB Status Badge** (showing total known CIs).
2. Click **Sync CMDB / Catalog**.
3. A banner notification confirms attributed count and exported shadow CIs.

**Via API:**
```powershell
$syncResponse = Invoke-RestMethod -Uri "https://shadowapi.corp.internal/api/v1/cmdb/sync" `
    -Method POST `
    -Headers @{ "Authorization" = "Bearer $AdminToken" }

$syncResponse | Format-List
```
Expected output:
```text
status           : success
attributed_count : 4
exported_shadows : 2
zombie_count     : 1
timestamp        : 2026-10-22T11:22:31Z
```

---

## 4. Circuit Breaker Troubleshooting

To prevent ServiceNow or Backstage network timeouts from blocking local API discovery:
- If 5 consecutive sync attempts to ServiceNow fail, the circuit breaker trips to **`OPEN`**.
- During `OPEN` state, scheduled syncs are deferred without throwing exceptions.
- To inspect breaker state:
```powershell
Invoke-RestMethod -Uri "https://shadowapi.corp.internal/api/v1/cmdb/status" -Method GET
```
If `circuit_breaker: "OPEN"`:
1. Verify ServiceNow instance reachability from the Windows Control Plane host: `Test-NetConnection -ComputerName "instance.service-now.com" -Port 443`.
2. Verify ServiceNow service account credentials.
3. The circuit breaker resets to **`HALF_OPEN`** automatically after 300 seconds and closes upon the next successful connection.
