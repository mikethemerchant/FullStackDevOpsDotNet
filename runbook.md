# IIS Setup Runbook for Windows Server 2022

**Date:** January 9, 2026  
**Phase:** Phase 1 - Core Infrastructure + AI Foundation  
**Task:** Configure IIS + ASP.NET Core Hosting Bundle

---

## 1. Verify IIS Installation Status

Check which IIS role services are installed:

```powershell
Get-WindowsFeature Web-Server,Web-Common-Http,Web-Static-Content,Web-Default-Doc,Web-Dir-Browsing,Web-Http-Errors,Web-Http-Redirect,Web-Health,Web-Http-Logging,Web-Log-Libraries,Web-Request-Monitor,Web-Http-Tracing,Web-Performance,Web-Stat-Compression,Web-Dyn-Compression,Web-Security,Web-Filtering,Web-Basic-Auth,Web-Windows-Auth,Web-App-Dev,Web-Net-Ext45,Web-Asp-Net45,Web-ISAPI-Ext,Web-ISAPI-Filter,Web-WebSockets,Web-Mgmt-Console | Select Name,InstallState
```

**Status as of January 9, 2026:**
- ✅ **Installed:** Web-Server, Static Content, Default Doc, HTTP Errors, HTTP Logging, Static/Dynamic Compression, WebSockets, Mgmt Console, Request Filtering
- ❌ **Missing (needed):** Web-Windows-Auth, Web-Http-Redirect, Web-Request-Monitor, Web-Http-Tracing, Web-Log-Libraries
- ❌ **Not needed for ASP.NET Core:** Web-Net-Ext45, Web-Asp-Net45, Web-ISAPI-Ext, Web-ISAPI-Filter (only needed for classic .NET Framework apps)

---

## 2. Install Required IIS Role Services

**IMPORTANT:** Run PowerShell as Administrator for this step.

Install the missing role services required for ASP.NET Core hosting with Windows Authentication:

```powershell
Install-WindowsFeature Web-Windows-Auth,Web-Http-Redirect,Web-Request-Monitor,Web-Http-Tracing,Web-Log-Libraries -IncludeManagementTools
```

**What each service provides:**
- **Web-Windows-Auth:** Enables Windows Integrated Authentication (Negotiate/Kerberos) for UI + API
- **Web-Http-Redirect:** HTTP redirect rules and URL rewrite support
- **Web-Request-Monitor:** Failed request tracing and diagnostics
- **Web-Http-Tracing:** Detailed HTTP request/response tracing for debugging
- **Web-Log-Libraries:** Advanced logging tools and ETW integration

**Expected result:** Success=True, RestartNeeded=No (typically no reboot required)

---

## 3. Verify ASP.NET Core Hosting Bundle

Check if ASP.NET Core Module V2 is installed:

```powershell
# Check installed .NET runtimes
dotnet --info

# Verify ANCM V2 module exists
Test-Path "C:\Program Files\IIS\Asp.Net Core Module\V2\aspnetcorev2.dll"

# Check IIS modules list for AspNetCoreModuleV2
Get-WebGlobalModule | Where-Object { $_.Name -like "*AspNetCore*" }
```

If ASP.NET Core Hosting Bundle is not installed:
1. Download from: https://dotnet.microsoft.com/download/dotnet/8.0 (Hosting Bundle)
2. Run installer
3. Reboot server after installation
4. Re-verify with commands above

---

## 4. Create Application Pool

### 4.1 GUI Method (IIS Manager)

1) Open IIS Manager: `Win + R` → `inetmgr` → Enter (or Server Manager → Tools → IIS Manager).
2) Left panel: expand the server → right-click **Application Pools** → **Add Application Pool...**
3) Settings (critical):
	- Name: `AspNetCore`
	- Managed runtime version: **No Managed Code** (do not pick v4.0)
	- Managed pipeline mode: `Integrated`
	- OK
4) Verify the pool shows **Status: Started**.

### 4.2 PowerShell Method (after GUI verified)

```powershell
$poolName = "AspNetCore"
New-WebAppPool -Name $poolName
Set-ItemProperty "IIS:\AppPools\$poolName" -Name managedRuntimeVersion -Value ""
Set-ItemProperty "IIS:\AppPools\$poolName" -Name managedPipelineMode -Value "Integrated"
Set-ItemProperty "IIS:\AppPools\$poolName" -Name processModel.identityType -Value "ApplicationPoolIdentity"
Set-ItemProperty "IIS:\AppPools\$poolName" -Name startMode -Value "AlwaysRunning"

# Verify
Get-IISAppPool -Name $poolName | Select-Object Name,ManagedRuntimeVersion,ManagedPipelineMode,State
```

Expected: ManagedRuntimeVersion is blank; ManagedPipelineMode Integrated; State Started.

---

## Status

- [X] Verified IIS installation status
- [X] Install missing IIS role services (all 5 services installed)
- [X] Verify ASP.NET Core Module (ANCM V2 + .NET 8.0.22 confirmed Jan 12)
- [X] Create and configure application pool (GUI completed)
- [ ] Create and configure application pool (PowerShell - optional automation)
- [ ] Create test site
- [ ] Enable Windows Authentication
- [ ] Test deployment

---

**Installation Results (January 12, 2026):**
✅ Verified:
- IIS Web-Server role
- ANCM V2 DLL present
- .NET 8.0.22 runtime
- AspNetCoreModuleV2 in IIS modules

**Next Action:** Optional PowerShell automation (Section 4.2) and create a test site to validate hosting.
