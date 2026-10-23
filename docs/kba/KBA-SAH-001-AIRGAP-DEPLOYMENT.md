# KBA-SAH-001: Air-Gapped Offline Cluster Deployment & Bootstrap

**Article ID:** KBA-SAH-001  
**Category:** Deployment & Setup  
**Severity:** Tier 3 (High)  
**Target Roles:** Platform Administrators, Infrastructure Leads, Enterprise Architects  
**Applies To:** Shadow API Hunter Enterprise v1.0.0+ (Windows Server 2022 / RHEL 9 / Ubuntu 22.04 LTS)  

---

## 1. Executive Summary & Objective

In high-assurance enterprise environments (banking, defense, healthcare), Shadow API Hunter must be installed without outbound internet connectivity or access to external package registries. This article guides platform administrators through the complete end-to-end bootstrap of:
1. Microsoft SQL Server Database with partitioned tables and RBAC roles.
2. Windows Management Control Plane (IIS UI + Windows Management API service).
3. Linux Distributed Telemetry Sensors via offline tarball bundles and answer files.
4. Cryptographic integrity verification via SHA-256 manifests.

---

## 2. Prerequisites & Pre-Flight Validation

Ensure the following assets from `dist/` are transferred via approved removable media or secure internal bastion to the target servers:
- `shadow-api-hunter-enterprise-v1.0.0.zip` (for Windows Control Plane)
- `shadow-api-hunter-enterprise-v1.0.0.tar.gz` (for Linux Fleet Nodes)
- `checksums.sha256` (cryptographic manifest)

### Hardware Sizing Verification
- **Windows Control Plane**: 4 vCPU, 16 GB RAM, 100 GB SSD storage, Windows Server 2019/2022, Microsoft SQL Server 2019/2022 Standard or Enterprise.
- **Linux Sensor Fleet Nodes**: 2-8 vCPU, 8-32 GB RAM, 200 GB SSD storage (`/var/spool/shadowapi` on high-IOPS NVMe).

---

## 3. Step-by-Step Deployment Procedure

### Phase A: Integrity Verification of Staged Packages

On both Windows and Linux hosts, verify package integrity against the canonical SHA-256 checksums before extraction.

**Windows PowerShell:**
```powershell
# Compute local hash of the offline release archive
$computedHash = (Get-FileHash -Path ".\shadow-api-hunter-enterprise-v1.0.0.zip" -Algorithm SHA256).Hash.ToLower()
Write-Host "Computed SHA-256: $computedHash"

# Compare against checksums.sha256 manifest
$expectedHash = (Get-Content ".\checksums.sha256" | Select-String "shadow-api-hunter-enterprise-v1.0.0.zip").Line.Split(" ")[0]
if ($computedHash -eq $expectedHash) {
    Write-Host "[SUCCESS] Cryptographic package integrity confirmed." -ForegroundColor Green
} else {
    Write-Error "[FATAL] Checksum mismatch! Corrupt or tampered media."
}
```

**Linux Bash:**
```bash
sha256sum shadow-api-hunter-enterprise-v1.0.0.tar.gz
# Ensure the hash matches the manifest entry
```

---

### Phase B: Microsoft SQL Server Schema Bootstrap

1. Log into the SQL Server Management Studio (SSMS) or connect via `sqlcmd`.
2. Create the dedicated database `ShadowApiHunter`:
```sql
CREATE DATABASE [ShadowApiHunter]
ON PRIMARY 
( NAME = N'ShadowApiHunter_Data', FILENAME = N'D:\SQLData\ShadowApiHunter.mdf', SIZE = 10GB, FILEGROWTH = 1GB )
LOG ON 
( NAME = N'ShadowApiHunter_Log', FILENAME = N'L:\SQLLog\ShadowApiHunter.ldf', SIZE = 2GB, FILEGROWTH = 512MB );
GO
ALTER DATABASE [ShadowApiHunter] SET RECOVERY FULL;
GO
```
3. Execute the schema migration scripts in strict numerical sequence:
```powershell
# Execute Initial Schema Migration
sqlcmd -S "sql-cluster.corp.internal" -d "ShadowApiHunter" -E -i ".\database\migrations\001_initial_schema.sql"

# Execute Enterprise RBAC Schema Migration
sqlcmd -S "sql-cluster.corp.internal" -d "ShadowApiHunter" -E -i ".\database\migrations\002_rbac_schema.sql"
```

---

### Phase C: Windows Control Plane Installation

1. Open PowerShell as Administrator.
2. Run the automated Windows installer script:
```powershell
.\windows\install-shadowapi-windows.ps1 `
    -InstallDir "C:\Program Files\ShadowAPI" `
    -SqlConnectionString "Server=sql-cluster.corp.internal;Database=ShadowApiHunter;Integrated Security=True;Encrypt=True;TrustServerCertificate=False;" `
    -WebSitePort 443 `
    -SslCertThumbprint "A1B2C3D4E5F6789012345678901234567890ABCD" `
    -ServiceAccount "CORP\svc-shadowapi"
```

3. Validate the Windows Management API service:
```powershell
Get-Service -Name "ShadowAPI.Management.Api" | Select-Object Status, StartType
Invoke-RestMethod -Uri "https://localhost/api/v1/health/ready" -Method GET
```

---

### Phase D: Linux Fleet Node Bootstrap

1. Extract the Linux fleet archive to `/tmp/shadowapi-install`:
```bash
mkdir -p /tmp/shadowapi-install
tar -xzf shadow-api-hunter-enterprise-v1.0.0.tar.gz -C /tmp/shadowapi-install
cd /tmp/shadowapi-install/shadow-api-hunter-enterprise-v1.0.0/linux
```

2. Select the unattended answer configuration file from `answers/` based on node role:
   - `core-sensor.conf` (Ingest, Normalizer, Spool, Session, Inventory)
   - `risk-worker.conf` (Risk Engine, Policy Cache, Active Verifier)
   - `sync-daemon.conf` (SIEM Streaming, Remediation, Evidence Vault)

3. Execute the hardened installer:
```bash
sudo bash ./shadow-api-hunter-linux-installer.sh \
    --answer-file ./answers/core-sensor.conf \
    --accept-defaults
```

4. Confirm that all enabled systemd units are running:
```bash
systemctl is-active shadowapi-collector shadowapi-ingest shadowapi-spool shadowapi-session shadowapi-inventory
```

---

## 4. Post-Deployment Verification Checklist

| Checkpoint | Target | Expected Output | Verification Command |
| :--- | :--- | :--- | :--- |
| **API Liveness** | Port 8080 (Windows) | `{"status":"live","version":"1.0.0"}` | `curl -k https://localhost:8080/health/live` |
| **Collector Liveness** | Port 9101 (Linux) | `{"status":"live","service":"shadowapi-collector"}` | `curl http://127.0.0.1:9101/health/live` |
| **Spool Queue** | Linux Host | Partition directory permission `0700` | `ls -ld /var/spool/shadowapi/queue` |
| **UI Air-Gap Safety** | IIS Web Root | 0 external CDN dependencies | `findstr /i "googleapis cdnjs unpkg" C:\inetpub\shadowapi\index.html` (Must be empty) |

---

## 5. Rollback & Emergency Uninstall

If installation fails:
- **Windows**: Run `.\windows\install-shadowapi-windows.ps1 -Uninstall` to remove IIS bindings, clean Windows services, and preserve database state.
- **Linux**: Run `sudo bash ./shadow-api-hunter-linux-installer.sh --purge` to stop all services, remove systemd unit symlinks, and archive `/var/spool/shadowapi/`.
