# ClassiGuard Shadow API Hunter — Enterprise API Security Platform (POC)

[![Release](https://img.shields.io/badge/Release-v1.0.0%20POC%20Evaluation-brightgreen.svg)]()
[![Windows Setup](https://img.shields.io/badge/Windows-Single%20Executable%20Setup%20(.exe)-blue.svg)]()
[![Linux Fleet](https://img.shields.io/badge/Linux-Single%20Tarball%20Fleet%20(.tar.gz)-orange.svg)]()
[![Runtime Dependency](https://img.shields.io/badge/Runtime%20Dependency-Zero%20(Pre--Compiled)-success.svg)]()
[![Code of Conduct](https://img.shields.io/badge/Code%20of%20Conduct-Indian%20Law%20%26%20Do's%20and%20Don'ts-blue.svg)](CODE_OF_CONDUCT.md)

---

> [!IMPORTANT]
> ### 6-Month Proof of Concept (POC) & Commercial Notice
> This software is published strictly as an open **Proof of Concept (POC) for a temporary evaluation period of six (6) months** (from October 2026 through April 2027) to gather community and enterprise feedback.
> **This product is NOT going to be free forever.** Following the conclusion of the 6-month evaluation phase, ClassiGuard reserves all rights to transition future versions and feature suites into commercial enterprise licensing, proprietary distribution tiers, and paid SLA support agreements.

> [!WARNING]
> ### Operator Liability & Law Enforcement Cooperation
> Anyone who downloads, installs, or executes this tool assumes **100% sole and exclusive legal and operational responsibility** for all actions, network traffic, system impacts, and data collected. ClassiGuard assumes **zero liability** for misuse, unauthorized scanning, or regulatory violations. ClassiGuard maintains a zero-tolerance policy against cyber offenses and **stands firmly with Law Enforcement Agencies** (including CERT-In and Indian Cyber Crime authorities).
> 
> Review our mandatory [**Code of Conduct & Acceptable Use Policy (Do's & Don'ts)**](CODE_OF_CONDUCT.md) before deployment.

---

## 1. Executive Summary & Capabilities

**ClassiGuard Shadow API Hunter** is an enterprise-grade API security platform designed for high-assurance enterprise environments (banking, defense, healthcare, telecommunications). The platform passively discovers and continuously inventories APIs from permitted network telemetry, reconciles them against declared OpenAPI specifications, detects undocumented **Shadow APIs** and deprecated **Zombie APIs**, and tracks risk findings across enterprise estates.

### Core Capabilities
- **Continuous API Inventory**: Live discovery explorer tracking endpoints, HTTP methods, authentication schemes, and sensitivity classifications.
- **Shadow & Zombie API Detection**: Automatic detection and triage of rogue endpoints and deprecated versions still receiving active traffic.
- **Interactive OpenAPI Drift Analyzer**: Visual diff analysis comparing observed production traffic against declared API specifications.
- **Zero-External-CDN Invariant**: Self-contained management application requiring zero external scripts, fonts, stylesheets, or CDNs (air-gap compliant).
- **Compliance & Audit Attestation**: Audit control mapping for PCI-DSS v4.0, SOC2 Type II, and NIST SP 800-53.

---

## 2. Pre-Compiled Ready-to-Run Packages (Zero Build / Zero Dependencies)

We ship direct, pre-compiled release artifacts with **zero runtime dependencies** (no Node.js, Python, or Rust required):

```
├── dist/
│   ├── ClassiGuard-Shadow-API-Hunter-Setup.exe          # Single Standalone Windows Setup Wizard (56.0 MB)
│   └── classiguard-shadow-api-hunter-linux-v1.0.0.tar.gz # Single Self-Contained Linux Sensor Fleet (48.7 MB)
└── packages/
    ├── proto/v1/                                        # Protobuf v3 Telemetry Contracts
    └── schemas/v1/                                      # JSON Schema Draft-07 Specifications
```

---

## 3. Windows Installation (Single Executable — Zero CLI)

For Windows 10, Windows 11, and Windows Server (2016/2019/2022/2025):

### Standard Setup Flow
1. Download [**`dist/ClassiGuard-Shadow-API-Hunter-Setup.exe`**](dist/ClassiGuard-Shadow-API-Hunter-Setup.exe).
2. **Double-click `ClassiGuard-Shadow-API-Hunter-Setup.exe`**.
3. Follow the interactive Setup Wizard:
   - Accept the **6-Month POC & Indian IT Act License Agreement**.
   - Select your destination directory (default `C:\Program Files\ClassiGuard\Shadow-API-Hunter\`).
   - Configure the Web Management Console port (default `8080`).
   - Enable SCM Service Resilience and Shortcuts.
   - Click **Install**.
4. **Done!** The installer automatically:
   - Installs the **ClassiGuard Enterprise Management Module** (`ClassiGuard.Management.exe`), Web Management Console, SQL migrations, and fleet orchestration packages.
   - Registers the service in Windows Service Control Manager with **Resilient Auto-Recovery** (automatically restarts within 5 seconds if terminated in Task Manager).
   - Auto-binds the native uninstaller in Windows **Programs and Features** (`appwiz.cpl`).
   - Launches the **Management Module Dashboard** and opens the Management Console at `http://localhost:8080/`.

### Standard Uninstallation
- Open Windows **Control Panel → Programs and Features** (or **Settings → Installed apps**).
- Select **ClassiGuard Shadow API Hunter Enterprise** and click **Uninstall**. All services, registry entries, and shortcuts are cleaned up automatically.

> [!NOTE]
> **No CLI typing, no PowerShell scripts, and no Node.js required.** The executable is 100% self-contained.

---

## 4. Linux Distributed Sensor Fleet (Single Self-Contained Tarball)

For Enterprise Linux (RHEL 8/9, AlmaLinux 9, Rocky Linux, Ubuntu 20.04/22.04 LTS, Debian 12):

### Steps
1. Download [**`dist/classiguard-shadow-api-hunter-linux-v1.0.0.tar.gz`**](dist/classiguard-shadow-api-hunter-linux-v1.0.0.tar.gz).
2. Extract the archive and run the included installer:
```bash
# Extract the single self-contained tarball
tar -xzf classiguard-shadow-api-hunter-linux-v1.0.0.tar.gz

# Run the installer (zero runtime dependencies)
sudo ./install.sh
```

### What the Linux Installer Configures Automatically
- Deploys pre-compiled static 64-bit binaries to `/opt/classiguard/bin/`.
- Provisions hardened non-root system service user (`shadowapi:shadowapi`).
- Registers and starts systemd services (`shadowapi-collector`, `shadowapi-ingest`, `shadowapi-spool`, `shadowapi-session`, `shadowapi-inventory`, `shadowapi-risk`, `shadowapi-policy`, `shadowapi-health`).
- Verifies node health at `http://127.0.0.1:9112/api/v1/health/node`.

---

## 5. Telemetry Contracts & Schemas

The repository provides standardized wire contracts and JSON schemas for passive API observation and asset modeling:

- **Protocol Buffer v3 Contracts** ([`packages/proto/v1/`](packages/proto/v1/)):
  - `api_observation.proto`: Privacy-redacted wire observation schema.
  - `api_asset.proto`: Inventory asset schema with taxonomy tags.
  - `finding.proto`: Risk finding and evidence proof schema.
  - `policy_snapshot.proto`: Cryptographically signed policy snapshot schema.
- **JSON Schema Specifications** ([`packages/schemas/v1/`](packages/schemas/v1/)):
  - Matching Draft-07 JSON schemas for API validation and message brokers.

---

## 6. Feedback & Community Participation (6-Month POC)

We actively solicit technical feedback, feature proposals, and bug discoveries from cybersecurity professionals and enterprise teams.

- **Submit Feedback**: Open a GitHub Issue labeled `[POC-Feedback]`.
- **Security Vulnerability Reporting**: Submit through **GitHub Private Security Advisories**.
- **Governance**: Refer to [**`CODE_OF_CONDUCT.md`**](CODE_OF_CONDUCT.md) for statutory terms and Acceptable Use requirements.

---

*ClassiGuard Shadow API Hunter — Enterprise API Security*
