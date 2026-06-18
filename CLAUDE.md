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

Both packages use **C# Script Tasks** (VSTA 17.0, .NET 4.7, `System.Net.Mail`) to send email notifications on completion — both a success mail and an OnError event-handler mail. All 4 Script Tasks read SMTP credentials from project parameters (see below) via `Dts.Variables["$Project::SMTPServer"]` etc.

## Connection Managers

Each package defines these connection managers (parameters override values at runtime via SSIS Catalog):

| Name | Type | Target |
|------|------|--------|
| `CINPSQL21` | ADO.NET (SQL) | `Data Source=10.9.57.8`, user `jdauser` |
| `CINPSQL21.NYC.AMLAW.CORP.jdauser` | OLE DB (SQLNCLI11.1) | Same server, used for legacy OLE DB tasks |

The OLE DB connection (`CINPSQL21.NYC.AMLAW.CORP.jdauser`) appears only in `Diamond360_Production_Deployment.dtsx`. A project-level `CINPSQL21.conmgr` connection manager file is also present.

## Project Parameters

`Project.params` defines three project-level parameters used by all mail Script Tasks in both packages:

| Parameter | Sensitive | Purpose |
|-----------|-----------|---------|
| `SMTPServer` | No | SMTP host (default: `email-smtp.us-east-1.amazonaws.com`) |
| `SMTPUserName` | No | SMTP auth username |
| `SMTPPassword` | Yes | SMTP auth password |

To change SMTP credentials, update the `Value` elements in `Project.params` — or override via SSIS Catalog environment mappings after deployment. Each parameter **must** have a unique GUID in its `ID` property; empty IDs cause a "Duplicate component name" load error.

When adding new project parameters to `Project.params`, always provide a unique GUID for the `<SSIS:Property SSIS:Name="ID">` field. Generate one with: `[guid]::NewGuid()` in PowerShell.

When adding project parameters to a Script Task, list them in the `ScriptProject` element's `ReadOnlyVariables` attribute (e.g. `$Project::SMTPServer`) **and** read them in C# via `Dts.Variables["$Project::SMTPServer"].Value.ToString()`. Both steps are required — omitting `ReadOnlyVariables` blocks runtime access.

## Sensitive Data / Protection Level

The project uses `EncryptSensitiveWithUserKey` protection level. Passwords and sensitive connection properties are encrypted to the **Windows user who last saved the project**. When opening the project on a different machine or user account, SSIS will prompt for sensitive values (passwords). For CI/CD or server deployment, use SSIS Catalog environment mappings to override sensitive parameters rather than storing them in the package.

## Ignored Files

`.gitignore` excludes `.vs/`, `bin/`, `obj/`, and `*.user` files. The compiled `.ispac` in `bin/Development/` is not tracked.
