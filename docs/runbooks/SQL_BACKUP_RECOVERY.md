# Microsoft SQL Server Backup & Disaster Recovery Runbook

**Version**: 1.0.0  
**Target Engine**: Microsoft SQL Server 2019 / 2022 (Standard or Enterprise)  
**Database**: `ShadowApiHunter`  
**Classification**: OPERATIONAL SRE / DBA RUNBOOK

---

## 1. Overview & Recovery Objectives

Shadow API Hunter Enterprise relies on Microsoft SQL Server as the authoritative central control plane data store for API inventory, risk evaluations, cryptographic policy snapshots, tenant configurations, and security audit logs.

### Recovery Objectives
- **Recovery Point Objective (RPO)**: $\le 15\text{ minutes}$ (standard); $\le 5\text{ minutes}$ (critical).
- **Recovery Time Objective (RTO)**: $\le 30\text{ minutes}$ for point-in-time recovery to a standby instance.
- **Database Recovery Model**: `FULL` (mandatory; `SIMPLE` recovery is prohibited in production).

---

## 2. Backup Strategy & Scheduling Matrix

```
       Sunday 01:00 UTC
      [ FULL BACKUP ] ---------------------------------------------> Weekly
             |
             +---> Monday 02:00 UTC [ DIFF BACKUP ]
             |          |
             |          +---> 02:15, 02:30, 02:45... [ LOG BACKUP ] (Every 15 mins)
             |
             +---> Tuesday 02:00 UTC [ DIFF BACKUP ]
                        |
                        +---> 02:15, 02:30, 02:45... [ LOG BACKUP ]
```

| Backup Type | Frequency | Retention | Compression | Verification | Target Path |
|---|---|---|---|---|---|
| **Full Backup** | Weekly (Sunday 01:00 UTC) | 30 days (local) / 365 days (cold) | `WITH COMPRESSION` | `WITH CHECKSUM` | `E:\SQLBackups\Full\` |
| **Differential** | Daily (MonΓÇôSat 02:00 UTC) | 14 days | `WITH COMPRESSION` | `WITH CHECKSUM` | `E:\SQLBackups\Diff\` |
| **Transaction Log**| Every 15 minutes (24/7) | 7 days | `WITH COMPRESSION` | `WITH CHECKSUM` | `E:\SQLBackups\Log\` |
| **Tail-Log Backup**| Pre-Recovery Disaster | Retained until drill closed | Standard | `WITH NORECOVERY` | `E:\SQLBackups\Tail\` |

---

## 3. Production Backup T-SQL Procedures

### 3.1 Weekly Full Database Backup
```sql
-- Schedule: Every Sunday at 01:00 UTC
BACKUP DATABASE [ShadowApiHunter]
TO DISK = N'E:\SQLBackups\Full\ShadowApiHunter_Full_INIT.bak'
WITH 
    FORMAT,
    INIT,
    NAME = N'ShadowApiHunter-Full Database Backup',
    COMPRESSION,
    CHECKSUM,
    STATS = 10;
GO

-- Verify physical backup integrity
RESTORE VERIFYONLY
FROM DISK = N'E:\SQLBackups\Full\ShadowApiHunter_Full_INIT.bak'
WITH CHECKSUM;
GO
```

### 3.2 Daily Differential Database Backup
```sql
-- Schedule: Monday through Saturday at 02:00 UTC
DECLARE @FileName NVARCHAR(512) = N'E:\SQLBackups\Diff\ShadowApiHunter_Diff_' + 
    CONVERT(NVARCHAR(30), GETDATE(), 112) + N'.bak';

BACKUP DATABASE [ShadowApiHunter]
TO DISK = @FileName
WITH 
    DIFFERENTIAL,
    FORMAT,
    INIT,
    NAME = N'ShadowApiHunter-Differential Backup',
    COMPRESSION,
    CHECKSUM,
    STATS = 10;
GO
```

### 3.3 15-Minute Transaction Log Backup
```sql
-- Schedule: Every 15 minutes
DECLARE @LogFile NVARCHAR(512) = N'E:\SQLBackups\Log\ShadowApiHunter_Log_' + 
    REPLACE(REPLACE(REPLACE(CONVERT(NVARCHAR(30), GETUTCDATE(), 120), '-', ''), ' ', '_'), ':', '') + N'.trn';

BACKUP LOG [ShadowApiHunter]
TO DISK = @LogFile
WITH 
    NOINIT,
    NAME = N'ShadowApiHunter-Transaction Log Backup',
    COMPRESSION,
    CHECKSUM,
    STATS = 10;
GO
```

---

## 4. Disaster Recovery & Point-In-Time Restore (PITR) Procedure

### 4.1 Step 1: Emergency Tail-Log Backup (Prevent Data Loss)
If the database engine or storage is damaged but log files are intact, capture the uncommitted transaction log tail:
```sql
USE master;
GO

BACKUP LOG [ShadowApiHunter]
TO DISK = N'E:\SQLBackups\Tail\ShadowApiHunter_TailLog.trn'
WITH 
    NO_TRUNCATE,
    NORECOVERY,
    CHECKSUM,
    STATS = 10;
GO
```
*Note: The database transitions to `RESTORING` state immediately, preventing further modifications.*

### 4.2 Step 2: Restore Last Valid Full Backup (with `NORECOVERY`)
```sql
USE master;
GO

RESTORE DATABASE [ShadowApiHunter]
FROM DISK = N'E:\SQLBackups\Full\ShadowApiHunter_Full_20261011.bak'
WITH 
    MOVE N'ShadowApiHunter_Data' TO N'D:\SQLData\ShadowApiHunter.mdf',
    MOVE N'ShadowApiHunter_Log'  TO N'L:\SQLLog\ShadowApiHunter_log.ldf',
    NORECOVERY,
    REPLACE,
    STATS = 10;
GO
```

### 4.3 Step 3: Restore Latest Differential Backup (with `NORECOVERY`)
```sql
RESTORE DATABASE [ShadowApiHunter]
FROM DISK = N'E:\SQLBackups\Diff\ShadowApiHunter_Diff_20261017.bak'
WITH 
    NORECOVERY,
    STATS = 10;
GO
```

### 4.4 Step 4: Restore Sequential Transaction Logs up to Point-in-Time
Apply all intermediate log files in chronological sequence with `NORECOVERY`:
```sql
RESTORE LOG [ShadowApiHunter]
FROM DISK = N'E:\SQLBackups\Log\ShadowApiHunter_Log_20261017_1400.trn'
WITH NORECOVERY;
GO

RESTORE LOG [ShadowApiHunter]
FROM DISK = N'E:\SQLBackups\Log\ShadowApiHunter_Log_20261017_1415.trn'
WITH NORECOVERY;
GO

-- Final transaction log restore with targeted STOPAT timestamp and RECOVERY
RESTORE LOG [ShadowApiHunter]
FROM DISK = N'E:\SQLBackups\Log\ShadowApiHunter_Log_20261017_1430.trn'
WITH 
    STOPAT = '2026-10-17 14:27:00.000',
    RECOVERY,
    STATS = 10;
GO
```

---

## 5. Post-Restore Verification & Integrity Validation

### 5.1 Run Comprehensive Database Consistency Check
```sql
DBCC CHECKDB (N'ShadowApiHunter')
WITH 
    NO_INFOMSGS,
    ALL_ERRORMSGS,
    DATA_PURITY;
GO
```
*Expected Result: `CHECKDB found 0 allocation errors and 0 consistency errors in database 'ShadowApiHunter'.`*

### 5.2 Validate Concurrency Tokens & Migrations Table
```sql
USE [ShadowApiHunter];
GO

-- 1. Verify schema migrations applied
SELECT migration_id, script_name, applied_at_utc, execution_time_ms
FROM operation.SchemaMigrations
ORDER BY migration_id;

-- 2. Verify multi-tenant partition counts
SELECT tenant_id, COUNT(*) AS asset_count
FROM inventory.ApiAssets
GROUP BY tenant_id;

-- 3. Verify ROWVERSION tokens intact
SELECT TOP 5 asset_id, tenant_id, route_path, row_version
FROM inventory.ApiAssets;
GO
```

### 5.3 Reconnect Windows Management API
1. Check Management API database connection string in `C:\ProgramData\ShadowAPI\config.yaml`:
   ```yaml
   database:
     server: "sql-cluster.corp.internal"
     database: "ShadowApiHunter"
     integrated_security: true
     connection_timeout_seconds: 15
   ```
2. Restart Windows Management service:
   ```powershell
   Restart-Service -Name "ShadowAPI.Management.Api"
   ```
3. Verify live readiness probe:
   ```powershell
   Invoke-RestMethod -Uri "http://localhost:8089/health/ready"
   ```

---

## 6. Always On Availability Groups (HA / DR Deployment)

For zero-downtime automated failover across redundant Windows Server nodes:

- **Primary Replica**: Read-Write, Synchronous Commit, Automatic Failover.
- **Secondary Synchronous Replica**: Read-Intent (offloads UI reporting queries), Synchronous Commit, Automatic Failover.
- **DR Asynchronous Replica**: Remote Datacenter, Asynchronous Commit, Manual Failover.
- **AG Listener DNS**: `sql-listener.corp.internal:1433`.

---

## 7. Operational SLA & Compliance Checklist

- [x] Full backups executed weekly with `CHECKSUM` and stored offsite.
- [x] Transaction logs backed up every 15 minutes; log truncation occurs only after successful backup.
- [x] PITR drills performed quarterly to validate RTO $\le 30\text{ minutes}$.
- [x] Database consistency check (`DBCC CHECKDB`) executed weekly without errors.
- [x] Multi-tenant data isolation verified across restored data tables.
