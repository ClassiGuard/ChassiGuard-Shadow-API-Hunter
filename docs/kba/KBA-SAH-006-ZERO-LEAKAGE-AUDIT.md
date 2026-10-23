# KBA-SAH-006: Conducting PCI-DSS Cardholder PAN & Credential Redaction Audits

**Article ID:** KBA-SAH-006  
**Category:** Compliance & Privacy Assurance  
**Severity:** Tier 2 (High Assurance)  
**Target Roles:** Compliance Auditors, Qualified Security Assessors (QSA), Privacy Officers, SRE  
**Applies To:** Shadow API Hunter Enterprise v1.0.0+ (Privacy Normalizer & Compliance Engine)  

---

## 1. Regulatory Context & Compliance Mandate

Under **PCI-DSS v4.0 (Requirement 3.4.1)** and **SOC2 Type II (CC6.1/CC6.8)**, any telemetry collection system monitoring cardholder data environments (CDE) must guarantee that:
1. Primary Account Numbers (PANs) are **never rendered readable** anywhere in telemetry logs, spool queues, or analytics databases.
2. Sensitive Authentication Data (SAD), API tokens, passwords, and authorization headers are **never transmitted to external systems or public cloud AI models**.

Shadow API Hunter implements multi-layer zero-leakage enforcement at the earliest point of ingest.

---

## 2. Multi-Layer Zero-Leakage Architecture

```text
[ Incoming Wire Traffic ]
           Γöé
           Γû╝
1. Collector Adapter (Sanitizer)  --> Luhn-checked PAN scrubbing ([REDACTED_CARD])
           Γöé                      --> Salted HMAC-SHA256 Client IP pseudonymization
           Γû╝
2. Rust Ingest Normalizer          --> Dynamic path folding (/card/{id}/charge)
           Γöé                      --> Zero-leakage validator (rejects plain Bearer/PANs)
           Γû╝
3. Encrypted Disk Spool           --> AES-256-GCM envelope encryption at rest
           Γöé
           Γû╝
4. AI Model Router                 --> Pre-inference deep-scrubbing (no egress)
           Γöé
           Γû╝
5. SQL Server Database Storage     --> Sanitized field names only; 0 literal payloads
```

---

## 3. Step-by-Step Compliance Audit Procedure

### Phase 1: Synthetic High-Risk Ingestion Test

Inject a synthetic test observation containing valid Luhn-calculated test card numbers and sensitive tokens to verify real-time redaction:

```bash
# Synthetic test payload with test Visa PAN (4532-0150-1234-5678)
curl -X POST http://127.0.0.1:9101/api/v1/collector/observations \
    -H "Content-Type: application/json" \
    -H "X-ShadowAPI-Source-ID: src-envoy-ingress-01" \
    -H "X-ShadowAPI-Source-Key: sec-envoy-token-2026" \
    -d '{
      "raw_path": "/api/v1/checkout/card/4532015012345678/verify?token=secret-token-xyz&cvv=999",
      "http_method": 2,
      "status_code": 200,
      "headers": {
        "Authorization": "Bearer confidential-bearer-token-12345"
      }
    }'
```
Expected HTTP response: `202 Accepted`.

---

### Phase 2: Direct Disk Spool Inspection

Verify that no plain cardholder data or tokens exist in the local encrypted spool files on the Linux node:

```bash
# Search active spool partitions for plain test PAN
if grep -r "4532015012345678" /var/spool/shadowapi/ 2>/dev/null; then
    echo "[CRITICAL VIOLATION] Raw PAN found on disk!"
    exit 1
else
    echo "[AUDIT PASS] Zero plain PAN strings found on disk."
fi

# Search for plain test token
if grep -r "secret-token-xyz" /var/spool/shadowapi/ 2>/dev/null; then
    echo "[CRITICAL VIOLATION] Raw token found on disk!"
    exit 1
else
    echo "[AUDIT PASS] Zero plain tokens found on disk."
fi

# Verify replacement placeholder
grep -r "REDACTED_CARD" /var/spool/shadowapi/queue/
# Output must show: /api/v1/checkout/card/[REDACTED_CARD]/verify?token=%5BREDACTED%5D
```

---

### Phase 3: SQL Server Database Store Inspection

On the Windows database host, query the persistent SQL Server inventory to ensure only normalized templates and sanitized metadata exist:

```sql
USE [ShadowApiHunter];
GO

-- 1. Assert zero card PANs in raw route strings
SELECT AssetId, RawPath, NormalizedRoute 
FROM [inventory].[ApiAssets]
WHERE RawPath LIKE '%4532%' OR NormalizedRoute LIKE '%4532%';
-- Must return 0 rows.

-- 2. Verify AuthSignal storage does not contain raw tokens
SELECT AssetId, AuthPresent, AuthScheme, TokenType, SubjectPseudonym
FROM [inventory].[ApiAssets]
WHERE SubjectPseudonym LIKE '%confidential%';
-- Must return 0 rows. (SubjectPseudonym is irreversible salted HMAC-SHA256).
```

---

### Phase 4: Exporting Signed Cryptographic Attestation

To generate official evidence for external PCI-DSS or SOC2 auditors:

1. In the IIS UI, navigate to **Tab 6: Compliance & Audit**.
2. Click **Export Audit Package (JSON)** to download the compliance bundle.
3. Query the cryptographic attestation endpoint:
```powershell
$attestation = Invoke-RestMethod -Uri "https://shadowapi.corp.internal/api/v1/compliance/attestation" -Method GET
$attestation | Format-List
```
Expected output:
```text
attestation_id   : att-pci-dss-v4-20261022
framework        : PCI-DSS v4.0 / SOC2 Type II
overall_status   : COMPLIANT
report_digest    : 3a7b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b
signature_hmac   : 9f8e7d6c5b4a3f2e1d0c9b8a7f6e5d4c3b2a1f0e9d8c7b6a5f4e3d2c1b0a9f8e
evidence_merkle  : e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
verified_at_utc  : 2026-10-22T11:22:31Z
```

---

## 4. Audit Evidence Archive

Save the exported `compliance-report-20261022.json` and signed attestation token in the compliance binder as definitive evidence for PCI-DSS Requirement 6.3.2 and Requirement 3.4.1 conformance.
