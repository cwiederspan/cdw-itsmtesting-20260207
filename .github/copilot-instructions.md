# Copilot Instructions

## Non Negotiables

- **DO NOT store sensitive information** — API keys, secrets, passwords, certificates, and connection strings must NEVER be stored in clear text in source code, configuration files, or anywhere that would be exposed via Git. Use secure alternatives such as environment variables, `dotnet user-secrets`, Azure Key Vault, GitHub Secrets, or equivalent mechanisms for your stack.
- **DO update this file** — Continually improve these instructions as the project evolves. Add project-specific context, architecture decisions, and implementation details so that Copilot (and future contributors) can be maximally effective.
- **DO include best practices and lessons learned** — Document what works, what doesn't, and why. This file should be a living record that grows smarter with every iteration.
- **DO validate changes** — Always run existing linters, builds, and tests before and after making changes. Never introduce regressions.
- **DO make minimal changes** — Prefer surgical, targeted edits over sweeping rewrites. Change only what is necessary to achieve the goal.

## Security Requirements

- **Authentication** — Prefer managed identities and `DefaultAzureCredential` over API keys wherever possible. Enterprise tenants may enforce `disableLocalAuth=true` on Azure resources, making API keys unavailable.
- **Secrets management** — Use `dotnet user-secrets` for local development, Azure Key Vault for deployed environments, and GitHub Secrets for CI/CD pipelines.
- **`.gitignore`** — Ensure sensitive files (`appsettings.Development.json`, `*.pfx`, `.env`, `secrets.json`) are excluded from source control.

## Nice to Haves

- Update command line tools (Azure CLI, .NET SDK, Node.js, etc.) before starting work on a new project.
- Ensure you have the latest version of GitHub Copilot CLI installed.
- When reproducing samples or quickstarts, check NuGet/npm package versions — samples often use outdated or pre-release versions with breaking API changes in newer releases.
- Prefer using ecosystem tools (`dotnet new`, `npm init`, `az` CLI, etc.) over manual file creation to reduce errors.
- Comment code only where clarification is needed — avoid obvious or redundant comments.

## Azure-Specific Guidance

- **Resource naming** — Use a consistent naming convention (e.g., `{prefix}-{service}-{date}`) for easy identification and cleanup.
- **Resource groups** — Create a dedicated resource group per project/experiment for clean teardown.
- **RBAC over keys** — Assign roles to managed identities and user principals rather than using shared API keys.
- **Managed Identity** — Enable System Assigned Managed Identity on resources that need to call other Azure services.
- **Azure CLI extensions** — Some Azure services require CLI extensions that prompt for installation. Run `az config set extension.use_dynamic_install=yes_without_prompt` to avoid interactive prompts during automation.

## Project Setup Conventions

- **Configuration** — Use `appsettings.json` for non-sensitive defaults and structure. Provide real values via environment variables or `dotnet user-secrets`.
- **Build & run commands** — Document the exact commands to restore, build, and run the project in the sections below.
- **Architecture** — Include a brief architecture diagram or description so that Copilot understands the data flow.

---

## Project Overview

### What It Does
This project restores an ITSM database from a `.bacpac` file into Azure SQL Database, then mirrors the tables into Microsoft Fabric for analytics. The workflow is executed in three sequential steps.

### Technology Stack
- **Azure CLI** — Resource provisioning and management
- **SqlPackage** — BACPAC import (v170.x required for Sql170 schema)
- **.NET 8** — Required runtime for SqlPackage
- **Microsoft Fabric** — Data mirroring destination

### Azure Resources Required

| Resource | Name | Notes |
|---|---|---|
| Resource Group | `cdw-itsmtesting-20260207-v01` | Location: `westus2` |
| Storage Account | `itsm20260207` | For bacpac staging, Entra auth only |
| Blob Container | `bacpaccontainer` | Holds uploaded ITSM.bacpac |
| Azure SQL Server | `itsm20260207v01` | Entra-only auth, System Assigned MI |
| Azure SQL Database | `ITSM` | General Purpose Serverless, no redundancy |

### Workflow Steps

#### Step 1: Create Azure SQL Infrastructure
1. Create resource group
2. Create storage account + blob container
3. Upload `ITSM.bacpac` to blob storage (using Entra auth)
4. Create Azure SQL Server (Entra-only, system managed identity)
5. Create empty database (General Purpose Serverless SKU)
6. Configure firewall (client IP + Azure services)
7. Assign RBAC roles for managed identity

#### Step 2: Import BACPAC via SqlPackage
1. Install SqlPackage 170.x and .NET 8
2. Obtain access token via `az account get-access-token --resource https://database.windows.net/`
3. Run `sqlpackage /Action:Import /SourceFile:./ITSM.bacpac /TargetServerName:itsm20260207v01.database.windows.net /TargetDatabaseName:ITSM /AccessToken:<token>`
4. Import takes ~55 minutes for 145MB bacpac — run in background

#### Step 3: Mirror to Microsoft Fabric
1. Configure Fabric mirroring for ITSM database tables
2. Details depend on Fabric workspace configuration

### Import Strategy
**`az sql db import` does NOT support Entra-only authentication.** Use SqlPackage instead:
1. Upload .bacpac to Azure Blob Storage for staging
2. Create empty database with serverless SKU
3. Use `sqlpackage /Action:Import` with `/AccessToken` for Entra auth
4. Import takes ~55 minutes for a 145MB bacpac file

---

## Lessons Learned & Best Practices

### Common Pitfalls

1. **Storage account key auth may be disabled by Azure Policy** — Use Entra auth with role assignments instead of storage keys. Use `--auth-mode login --as-user` for blob SAS tokens.
2. **`az sql db import` does NOT work with Entra-only auth** — It requires SQL admin password. Use SqlPackage with `/AccessToken` instead.
3. **SqlPackage version must match bacpac schema version** — SQL Server 2022 bacpac (Sql170) requires SqlPackage 170.x.
4. **SqlPackage 170.2.70 requires .NET 8.0.20** — Use SqlPackage 170.0.94 if you only have .NET 8.0.15.
5. **Role assignments take ~60 seconds to propagate** — Wait before using newly assigned permissions.
6. **User delegation SAS tokens work when key auth is disabled** — Use `--auth-mode login --as-user` for blob SAS.
7. **Import of 145MB bacpac takes ~55 minutes** — Run in background and plan accordingly.
8. **Azure CLI extensions may prompt for installation** — Run `az config set extension.use_dynamic_install=yes_without_prompt` first.

### Success Markers

- [ ] Resource group created
- [ ] Storage account and container created
- [ ] ITSM.bacpac uploaded to blob storage
- [ ] SQL Server created with Entra-only auth and system MI
- [ ] Database created with serverless SKU
- [ ] Firewall configured
- [ ] RBAC roles assigned
- [ ] SqlPackage installed (v170.x)
- [ ] BACPAC imported successfully
- [ ] Fabric mirroring configured
