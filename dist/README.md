# ClassiGuard Shadow API Hunter — Distribution Deliverables

This directory contains official pre-compiled release artifacts for ClassiGuard Shadow API Hunter Enterprise. All packages are self-contained and pre-configured for direct, zero-runtime-dependency deployment.

## Packages Included

1. **`ClassiGuard-Shadow-API-Hunter-Setup.exe`** (Windows Setup Wizard)
   - Interactive GUI Setup Wizard for Windows 10, Windows 11, and Windows Server (2016/2019/2022/2025).
   - Installs the unified **ClassiGuard Enterprise Management Module** (`ClassiGuard.Management.exe`), Web Management Console, SQL migrations, and fleet deployment packages.
   - Automatically registers in Windows **Programs and Features** (`appwiz.cpl`) with auto-bound uninstaller.
   - Configures Windows SCM **Resilient Auto-Recovery** (auto-restart within 5s if terminated from Task Manager).

2. **`classiguard-shadow-api-hunter-linux-v1.0.0.tar.gz`** (Linux Sensor Fleet)
   - Complete pre-compiled 64-bit native binary bundle for Enterprise Linux (RHEL 8/9, Ubuntu 20.04/22.04 LTS, Debian 12).
   - Contains all 12 sensor microservices, systemd service units, and automated `install.sh`.
   - Operates with zero runtime dependencies (no Node.js, Python, or Rust required).

---
*ClassiGuard Shadow API Hunter — Enterprise API Security*
