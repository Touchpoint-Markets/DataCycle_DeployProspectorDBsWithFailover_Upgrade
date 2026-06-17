# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **SQL Server Integration Services (SSIS)** project that deploys ProspectorDB data with failover support. It targets SQL Server 2025 and is built/edited using **Visual Studio 2022 with SQL Server Data Tools (SSDT) 17.x**.

## Build and Deploy

**Build in Visual Studio:**
Open `DataCycle_DeployProspectorDBsWithFailover_Upgrade.slnx` in Visual Studio 2022 with the SSDT extension. Build produces `bin/Development/DeployProspectorDBsWithFailover.ispac`.

**Build via MSBuild (command line):**
```powershell
msbuild DataCycle_DeployProspectorDBsWithFailover_Upgrade.dtproj /p:Configuration=Development
```

**Deploy the `.ispac` to SSIS Catalog:**
Use the SSIS Deployment Wizard (`ISDeploymentWizard.exe`) or SQL Server Management Studio to deploy `bin/Development/DeployProspectorDBsWithFailover.ispac` to the SSISDB catalog on the target server.

## Architecture

The project contains two entry-point SSIS packages:

### `Diamond360_Production_Deployment.dtsx`
Executes the T-SQL stored procedure `Diamond360.dbo.MOVE_APIDATA_QATOLIVE` on the production SQL Server to promote QA data to live, then sends a completion notification email via the C# Script Task.

### `Diamond360_RawData_Production_Deployment.dtsx`
Handles raw data deployment for the Diamond360 database, followed by a completion notification email via C# Script Task.

Both packages use **C# Script Tasks** (VSTA 17.0, .NET, `System.Net.Mail`) to send email notifications on completion.

## Connection Managers

Each package defines these connection managers (parameters override values at runtime via SSIS Catalog):

| Name | Type | Target |
|------|------|--------|
| `CINPSQL21` | ADO.NET (SQL) | `Data Source=10.9.57.8`, user `jdauser` |
| `CINPSQL21.NYC.AMLAW.CORP.jdauser` | OLE DB (SQLNCLI11.1) | Same server, used for legacy OLE DB tasks |
| `SMTP Connection Manager` | SMTP | `smtp-relay.gmail.com`, SSL, Windows auth |

The OLE DB connection (`CINPSQL21.NYC.AMLAW.CORP.jdauser`) appears only in `Diamond360_Production_Deployment.dtsx`.

## Sensitive Data / Protection Level

The project uses `EncryptSensitiveWithUserKey` protection level. Passwords and sensitive connection properties are encrypted to the **Windows user who last saved the project**. When opening the project on a different machine or user account, SSIS will prompt for sensitive values (passwords). For CI/CD or server deployment, use environment variables or SSIS Catalog parameters to override sensitive connection strings rather than storing them in the package.

## Project Parameters

`Project.params` is currently empty — all connection overrides are package-level parameters defined in the project manifest (`DataCycle_DeployProspectorDBsWithFailover_Upgrade.dtproj` under `<SSIS:DeploymentInfo>`). These are configured post-deployment via SSIS Catalog environment mappings.

## Ignored Files

`.gitignore` excludes `.vs/`, `bin/`, `obj/`, and `*.user` files. The compiled `.ispac` in `bin/Development/` is not tracked.
