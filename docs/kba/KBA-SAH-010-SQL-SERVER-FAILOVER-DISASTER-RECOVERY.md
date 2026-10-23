# KBA-SAH-010: SQL Server Disaster Recovery, Always On Failover & Tail-Log Restore

**Article ID:** KBA-SAH-010  
**Category:** High Availability & Database Operations  
**Severity:** Tier 3 (Critical Disaster Recovery)  
**Target Roles:** Database Administrators (DBA), Infrastructure SRE, Platform Administrators  
**Applies To:** Microsoft SQL Server 2019/2022 (ShadowApiHunter Database)  

---

## 1. Architecture Overview & High Availability Strategy

Shadow API Hunter's control plane depends strictly on **Microsoft SQL Server** per **ADR-0002**. Production enterprise deployments utilize:
- **Always On Availability Groups (AG)**: Multi-subnet synchronous commit with automatic failover between Primary (`SQL-NODE-01`) and Secondary (`SQL-NODE-02`).
- **Full Recovery Model**: Full weekly backup, daily differential, and 15-minute transaction log backups (`.trn`).
- **MultiSubnetFailover**: Windows Management API connection strings configured with `MultiSubnetFailover=True;` to redirect seamlessly upon replica role transition.

---

## 2. Scenario 1: Planned Always On Maintenance Failover

Execute before patching the Primary SQL host:

1. Connect to the Availability Group Listener via SSMS or `sqlcmd`:
```powershell
sqlcmd -S "sql-ag-listener.corp.internal" -E -Q "SELECT replica_server_name, role_desc, synchronization_state_desc FROM sys.dm_hadr_availability_replica_states;"
```
Confirm secondary is `SYNCHRONIZED`.

2. On the target Secondary replica (`SQL-NODE-02`), execute graceful failover:
```sql
ALTER AVAILABILITY GROUP [AG_ShadowApiHunter] FAILOVER;
GO
```

3. Confirm that `SQL-NODE-02` transitioned to `PRIMARY_NORMAL`.
4. Validate Windows Management API connectivity:
```powershell
Invoke-RestMethod -Uri "https://shadowapi.corp.internal/api/v1/health/ready" -Method GET
# Expected: {"status":"ready","database":"connected","storage":"healthy"}
```

---

## 3. Scenario 2: Unplanned Primary Crash & Forced Failover

If the Primary host experiences catastrophic hardware failure and the AG does not automatically fail over:

1. Connect to the secondary host (`SQL-NODE-02`) as sysadmin.
2. Force failover with data loss assessment:
```sql
ALTER AVAILABILITY GROUP [AG_ShadowApiHunter] FORCE_FAILOVER_ALLOW_DATA_LOSS;
GO
```
3. Resume synchronization across databases:
```sql
ALTER DATABASE [ShadowApiHunter] SET HADR RESUME;
GO
```

---

## 4. Scenario 3: Corrupt Database & Point-In-Time Restore (PITR)

If database files are corrupted or an accidental administrative operation occurred:

### Step 1: Capture the Emergency Tail-Log Backup
If the primary transaction log (`.ldf`) is still accessible, capture the tail-log to prevent loss of recent transactions:
```sql
BACKUP LOG [ShadowApiHunter] 
TO DISK = N'B:\SQLBackups\ShadowApiHunter_TailLog.trn' 
WITH NO_TRUNCATE, NOFORMAT, INIT, NAME = N'ShadowApiHunter-TailLog';
GO
```

### Step 2: Restore Full Backup (WITH NORECOVERY)
```sql
RESTORE DATABASE [ShadowApiHunter]
FROM DISK = N'B:\SQLBackups\ShadowApiHunter_Full_Weekly.bak'
WITH NORECOVERY, REPLACE;
GO
```

### Step 3: Restore Differential Backup (WITH NORECOVERY)
```sql
RESTORE DATABASE [ShadowApiHunter]
FROM DISK = N'B:\SQLBackups\ShadowApiHunter_Diff_Daily.bak'
WITH NORECOVERY;
GO
```

### Step 4: Restore Transaction Logs up to Point-in-Time (WITH RECOVERY)
```sql
RESTORE LOG [ShadowApiHunter]
FROM DISK = N'B:\SQLBackups\ShadowApiHunter_Log_1000.trn'
WITH NORECOVERY;
GO

-- Restore Tail-Log and bring database online
RESTORE LOG [ShadowApiHunter]
FROM DISK = N'B:\SQLBackups\ShadowApiHunter_TailLog.trn'
WITH STOPAT = '2026-10-22 10:45:00', RECOVERY;
GO
```

### Step 5: Execute Consistency Check
```sql
DBCC CHECKDB ([ShadowApiHunter]) WITH NO_INFOMSGS, ALL_ERRORMSGS;
GO
```

---

## 5. Post-Disaster Service Health Validation

Restart the Windows Control Plane services to re-establish connection pools:

```powershell
Restart-Service -Name "ShadowAPI.Management.Api"
Restart-Service -Name "ShadowAPI.Management.Jobs"
Restart-Service -Name "ShadowAPI.Management.Sync"

# Run master verification hook
& "C:\Program Files\ShadowAPI\scripts\verify-checkpoint.ps1"
```
Ensure all 15 invariant tests report `[PASS]`.
