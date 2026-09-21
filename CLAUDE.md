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

Both packages use **C# Script Tasks** (VSTA 17.0, .NET 4.7, `System.Net.Mail`) to send email notifications on completion — both a success mail and an OnError event-handler mail. All 4 Script Tasks read `$Project::SMTPServer`/`$Project::SMTPFromAddress`/`$Project::sendEmailFlag` from project parameters and retrieve the SMTP username/password from AWS Secrets Manager at runtime — see "AWS Secrets Manager Credential Retrieval" below.

**Script Tasks that currently have no `<BinaryItem>`** (must be built in SSDT before deploying): all 4 mail-sending Script Tasks — `Send Mail Diamond360_Production_Deployment Completed` and `Send Mail on Error` in `Diamond360_Production_Deployment.dtsx`, `Send Mail Diamond360_RawData_Production_Deployment Completed` and `Send Mail on Error` in `Diamond360_RawData_Production_Deployment.dtsx` — had their `<BinaryItem>` deleted after switching to AWS Secrets Manager credential retrieval. One SSDT build per package regenerates both of that package's missing BinaryItems.

## Connection Managers

Each package defines these connection managers (parameters override values at runtime via SSIS Catalog):

| Name | Type | Target |
|------|------|--------|
| `CINPSQL21` | ADO.NET (SQL) | `Data Source=10.9.57.8`, user `jdauser` |
| `CINPSQL21.NYC.AMLAW.CORP.jdauser` | OLE DB (SQLNCLI11.1) | Same server, used for legacy OLE DB tasks |

The OLE DB connection (`CINPSQL21.NYC.AMLAW.CORP.jdauser`) appears only in `Diamond360_Production_Deployment.dtsx`. A project-level `CINPSQL21.conmgr` connection manager file is also present.

## Project Parameters

`Project.params` defines five project-level parameters used by all 4 mail Script Tasks across both packages:

| Parameter | Sensitive | Purpose |
|-----------|-----------|---------|
| `SMTPServer` | No | SMTP host (default: `email-smtp.us-east-1.amazonaws.com`) |
| `AwsSecretName` | No | Name of the AWS Secrets Manager secret holding the SMTP `UserName`/`Password` JSON (default `JudyDiamond-SMTP`), retrieved at runtime — see "AWS Secrets Manager Credential Retrieval" below |
| `AwsRegion` | No | AWS region of the `AwsSecretName` secret (default `us-east-1`) |
| `SMTPFromAddress` | No | `mail.From` sender address |
| `sendEmailFlag` | No | Master on/off switch (Int32) for every `.Send()` call — `1` = send, `0` = skip |

`SMTPUserName`/`SMTPPassword` were **removed** — they held a live AWS SES access key and password in plaintext (`Sensitive=0`, checked into git). Credentials now live only in AWS Secrets Manager, never in the project or any exported `.ispac`/environment mapping.

To rotate SMTP credentials, update the secret in AWS Secrets Manager — no project redeploy needed. To change `SMTPServer`/`SMTPFromAddress`/`sendEmailFlag`, update the `Value` elements in `Project.params` — or override via SSIS Catalog environment mappings after deployment. Each parameter **must** have a unique GUID in its `ID` property; empty IDs cause a "Duplicate component name" load error.

When adding new project parameters to `Project.params`, always provide a unique GUID for the `<SSIS:Property SSIS:Name="ID">` field. Generate one with: `[guid]::NewGuid()` in PowerShell.

When adding project parameters to a Script Task, list them in the `ScriptProject` element's `ReadOnlyVariables` attribute (e.g. `$Project::SMTPServer`) **and** read them in C# via `Dts.Variables["$Project::SMTPServer"].Value.ToString()`. Both steps are required — omitting `ReadOnlyVariables` blocks runtime access.

## AWS Secrets Manager Credential Retrieval

SMTP credentials are stored as a JSON secret in AWS Secrets Manager, named by `$Project::AwsSecretName` (default `JudyDiamond-SMTP`) in region `$Project::AwsRegion` (default `us-east-1`). The secret value is JSON: `{"Host":"...","Port":587,"UserName":"...","Password":"...","EnableSsl":true}` — only `UserName`/`Password` are used; `Host`/`Port`/`EnableSsl` are ignored (the project keeps `$Project::SMTPServer` and the hardcoded port 587 as-is).

**Each of the 4 mail-sending Script Tasks embeds its own copy** of two private static helpers:

```csharp
private static string GetSecretString(string secretName, string region)
{
    using (var client = new Amazon.SecretsManager.AmazonSecretsManagerClient(Amazon.RegionEndpoint.GetBySystemName(region)))
    {
        var response = client.GetSecretValue(new Amazon.SecretsManager.Model.GetSecretValueRequest { SecretId = secretName });
        return response.SecretString;
    }
}

private static string ExtractJsonStringField(string json, string fieldName)
{
    var match = System.Text.RegularExpressions.Regex.Match(json, "\"" + fieldName + "\"\\s*:\\s*\"((?:[^\"\\\\]|\\\\.)*)\"");
    if (!match.Success)
        throw new System.Exception("Field '" + fieldName + "' not found in secret.");
    return match.Groups[1].Value.Replace("\\\"", "\"").Replace("\\\\", "\\");
}
```

`ExtractJsonStringField` is a hand-rolled regex extractor, not a JSON library — avoids adding Newtonsoft.Json as a dependency for a 2-field read.

**AWS credentials for calling Secrets Manager**: `AmazonSecretsManagerClient(RegionEndpoint)` uses the AWS SDK's default credential provider chain (environment variables, `~/.aws/credentials`, or — the expected case — an IAM role/instance profile on the machine running the SSIS Catalog). No access key/secret is stored anywhere in the project; the server must have `secretsmanager:GetSecretValue` permission on the `JudyDiamond-SMTP` secret via its IAM role.

**SDK version**: `AWSSDK.Core`/`AWSSDK.SecretsManager` 3.7.500, targeting net45 (not AWS SDK v4) — the net45 build of both packages is fully self-contained (zero further NuGet dependencies), so it drops into `Imports\` with just 2 DLLs.

**None of the 4 tasks had any prior external-DLL dependency**, so each needed the full `static ScriptMain()` `AssemblyResolve` handler + `private static string _importsDir` field + `Main()`/`RunMain()` split (`Main()` sets `_importsDir` from `User::ImportsPath` then calls `RunMain()`; `RunMain()` holds the original body, with the credential-fetch calls added). This ordering is required because `GetSecretString`'s method body is the only place the `Amazon.*` types are referenced, and its JIT (triggered at first call, inside `RunMain()`) must happen after `_importsDir` is populated. `User::ImportsPath` is a new package variable in both packages, default `I:\Git Solutions\DataCycle_DeployProspectorDBsWithFailover_Upgrade\Imports\`.

**`Imports\` folder** (new, checked into git — not excluded by `.gitignore`) contains `AWSSDK.Core.dll` and `AWSSDK.SecretsManager.dll` (v3.7.500, net45). Each embedded `.csproj` references both via `HintPath` into this folder:
```xml
<Reference Include="AWSSDK.Core, Version=3.3.0.0, Culture=neutral, PublicKeyToken=885c28607f98e604, processorArchitecture=MSIL">
  <HintPath>I:\Git Solutions\DataCycle_DeployProspectorDBsWithFailover_Upgrade\Imports\AWSSDK.Core.dll</HintPath>
</Reference>
<Reference Include="AWSSDK.SecretsManager, Version=3.3.0.0, Culture=neutral, PublicKeyToken=885c28607f98e604, processorArchitecture=MSIL">
  <HintPath>I:\Git Solutions\DataCycle_DeployProspectorDBsWithFailover_Upgrade\Imports\AWSSDK.SecretsManager.dll</HintPath>
</Reference>
```
(`Version=3.3.0.0` is the assembly's actual `AssemblyVersion` — AWS SDK assemblies keep a stable low `AssemblyVersion` across NuGet package versions; 3.7.500 is the real package version, encoded in the `AssemblyResolve` NuGet-cache fallback path.)

## Sensitive Data / Protection Level

The project uses `EncryptSensitiveWithUserKey` protection level. Passwords and sensitive connection properties are encrypted to the **Windows user who last saved the project**. When opening the project on a different machine or user account, SSIS will prompt for sensitive values (passwords). For CI/CD or server deployment, use SSIS Catalog environment mappings to override sensitive parameters rather than storing them in the package.

## Ignored Files

`.gitignore` excludes `.vs/`, `bin/`, `obj/`, and `*.user` files. The compiled `.ispac` in `bin/Development/` is not tracked. `Imports\*.dll` **is** tracked (checked in as binary) — it is not excluded by `.gitignore`.
