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
- **Azure Container Instances (ACI)** — Ephemeral compute for running the bacpac import inside Azure
- **Azure Container Registry (ACR)** — Temporary image registry for the import container (deleted after use)
- **SqlPackage** — BACPAC import tool (runs inside the ACI container)
- **Microsoft Fabric** — Data mirroring destination

### Azure Resources (Persistent)

| Resource | Name | Notes |
|---|---|---|
| Resource Group | `cdw-itsmtesting-20260207-v01` | Location: `westus2` |
| Azure SQL Server | `itsm20260207v01` | Entra-only auth, System Assigned MI |
| Azure SQL Database | `ITSM` | General Purpose Serverless (GP_S_Gen5_1), no redundancy |

### Azure Resources (Ephemeral — created during import, deleted after)

| Resource | Name | Purpose |
|---|---|---|
| Container Registry | `itsm20260207acr` | Hosts the import container image |
| Container Instance | `bacpac-import` | Runs SqlPackage inside Azure |
| User-Assigned MI | `bacpac-import-identity` | Authenticates ACI to SQL Server |

### Workflow Steps

#### Step 1: Create Azure SQL Infrastructure
1. Run `az config set extension.use_dynamic_install=yes_without_prompt`
2. Create resource group (`cdw-itsmtesting-20260207-v01` in `westus2`)
3. Create Azure SQL Server (`itsm20260207v01`) with:
   - Entra-only authentication (`--enable-ad-only-auth`)
   - System Assigned Managed Identity (`--assign-identity --identity-type SystemAssigned`)
   - Current user as AD admin (`--external-admin-principal-type User`)
4. Create empty database (`ITSM`) with:
   - General Purpose Serverless SKU (`--edition GeneralPurpose --compute-model Serverless --family Gen5 --capacity 1`)
   - Auto-pause after 60 min (`--auto-pause-delay 60`)
   - Local backup redundancy (`--backup-storage-redundancy Local`)
5. Configure firewall rules:
   - Allow current client IP (`curl -s https://ifconfig.me`)
   - Allow Azure services (`0.0.0.0` to `0.0.0.0`)

#### Step 2: Import BACPAC via ACI + SqlPackage
This is the proven approach. Do NOT try to run SqlPackage locally — it will likely fail due to token expiry over slow networks.

1. **Create ephemeral resources:**
   - Create a user-assigned managed identity (`bacpac-import-identity`)
   - Set the MI as SQL Server AD admin (note: this temporarily removes your user as admin)
   - Wait 60 seconds for the admin assignment to propagate
   - Create an ACR (`itsm20260207acr`, Basic SKU, admin enabled)

2. **Build the import container:**
   - Create a build context directory with:
     - `Dockerfile` — Based on `mcr.microsoft.com/dotnet/runtime:8.0`, installs SqlPackage (via `https://aka.ms/sqlpackage-linux`) and Azure CLI, copies the bacpac file
     - `import.sh` — Logs in via managed identity (`az login --identity --client-id <CLIENT_ID>`), gets an access token for `https://database.windows.net/`, runs `sqlpackage /Action:Import`
     - `ITSM.bacpac` — The bacpac file
   - Build using ACR Tasks: `az acr build --registry <acr> --image bacpac-import:latest .`

3. **Run the import:**
   - Create ACI with: `--assign-identity <MI_RESOURCE_ID>`, `--os-type Linux`, `--cpu 2 --memory 4`, `--restart-policy Never`
   - Pass environment variables: `SQL_SERVER`, `SQL_DATABASE`, `MI_CLIENT_ID`
   - Monitor with: `az container logs --name bacpac-import --resource-group <rg>`
   - Expected completion: ~21 minutes for 145MB bacpac on serverless SKU

4. **Clean up after success:**
   - Restore original user as SQL Server AD admin
   - Delete ACI: `az container delete --name bacpac-import`
   - Delete ACR: `az acr delete --name itsm20260207acr`
   - Delete managed identity: `az identity delete --name bacpac-import-identity`

#### Step 3: Mirror to Microsoft Fabric
1. Configure Fabric mirroring for ITSM database tables
2. Details depend on Fabric workspace configuration

---

## Key Design Decisions

### Why ACI instead of running SqlPackage locally?
- **`az sql db import` does NOT support Entra-only authentication** — It requires a SQL admin password, which is unavailable when Entra-only auth is enforced.
- **SqlPackage only reads local files** — It cannot import directly from Azure Blob Storage URLs, so blob staging adds no value.
- **Azure AD access tokens expire in ~75-87 minutes** — Running SqlPackage locally over home internet takes ~90+ min for a 145MB bacpac, exceeding the token lifetime. From a GitHub Codespace it takes ~45-55 min (tight but possible).
- **ACI runs inside Azure's network** — The same import completes in ~21 minutes on the cheapest serverless SKU, well within token lifetime. No need to scale up the database.

### Why a user-assigned managed identity?
- ACI with system-assigned MI gets a new principal ID each time it's recreated, requiring the SQL admin to be updated each time.
- A user-assigned MI keeps a stable principal ID across ACI delete/recreate cycles, which is essential since ACI with `--restart-policy Never` cannot be restarted.

### Why not use an Entra ID group as SQL admin?
- This would be the ideal approach: create a group, add both your user and the MI, set the group as admin. This avoids swapping the admin back and forth.
- Not implemented yet — would be an improvement for future iterations.

---

## Lessons Learned & Best Practices

### Import Strategy
1. **ACI is the recommended approach for bacpac import** — Package SqlPackage + bacpac into a Docker image, run as ACI in the same Azure region as the SQL server. Import completes in ~21 min on serverless SKU.
2. **SqlPackage cannot resume a failed import** — A failed import leaves the database partially populated. You must `az sql db delete` and `az sql db create` before retrying.
3. **SQL admin changes take ~60 seconds to propagate** — Always wait after updating the AD admin before the ACI attempts to connect.
4. **Use `--client-id` (not `--username`) for user-assigned MI** — `az login --identity --username` is deprecated.
5. **ACI with `--restart-policy Never` cannot be restarted** — Must delete and recreate. Use a user-assigned MI so the identity persists.

### Azure SQL
6. **Entra-only auth blocks `az sql db import`** — This command requires SQL auth (username/password). Use SqlPackage with `/AccessToken` instead.
7. **Azure AD tokens expire in ~75-87 minutes** — Ensure the import can complete within this window. ACI approach handles this easily.
8. **SQL Server supports one AD admin (user or group)** — Use an Entra group to avoid swapping admins. The group can contain both human users and managed identities.

### SqlPackage
9. **SqlPackage version must match bacpac schema** — SQL Server 2022 bacpac (Sql170) requires SqlPackage 170.x.
10. **Use the standalone Linux zip for containers** — Download from `https://aka.ms/sqlpackage-linux` rather than the .NET tool install.
11. **SqlPackage only reads local files** — No support for Azure Blob Storage URLs as source.

### General Azure
12. **Run `az config set extension.use_dynamic_install=yes_without_prompt`** — Prevents interactive prompts for CLI extension installs.
13. **Register resource providers before first use** — `Microsoft.ContainerRegistry` and `Microsoft.ContainerInstance` may not be registered on the subscription. Use `az provider register --namespace <ns> --wait`.
14. **Blob storage is unnecessary for this workflow** — Since SqlPackage can't use blob URLs and `az sql db import` can't use Entra-only auth, staging the bacpac in blob storage adds no value.

### Success Markers

- [x] Resource group created
- [x] SQL Server created with Entra-only auth and system MI
- [x] Database created with serverless SKU
- [x] Firewall configured (client IP + Azure services)
- [x] BACPAC imported successfully via ACI (~21 min)
- [x] Ephemeral resources cleaned up (ACR, ACI, MI)
- [ ] Fabric mirroring configured
