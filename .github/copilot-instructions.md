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
| Entra ID Group | `itsm-sql-admins` | SQL admin group — **keep this even when deleting resource groups** |
| Fabric Workspace | `My ITSM Workspace` | ID: `6a55d22a-ae85-4707-89fd-ec4e0e261d17` — **keep across teardowns** |
| Fabric Connection | `ITSM-AzureSQL-Connection` | ID: `76f3ff4c-19e2-4070-b0c8-cb6b478fa198` — recreate if SQL server changes |
| Fabric Capacity | `itsmfabric20260207` | F2 SKU, West US 2 — **lives in the resource group** |
| Resource Group | `cdw-itsmtesting-20260207-v02` | Location: `westus2` |
| Azure SQL Server | `itsm20260207v02` | Entra-only auth, System Assigned MI |
| Azure SQL Database | `ITSM` | General Purpose Serverless (GP_S_Gen5_1), no redundancy |

### Azure Resources (Ephemeral — created during import, deleted after)

| Resource | Name | Purpose |
|---|---|---|
| Container Registry | `itsm20260207v02acr` | Hosts the import container image |
| Container Instance | `bacpac-import` | Runs SqlPackage inside Azure |
| User-Assigned MI | `bacpac-import-identity` | Authenticates ACI to SQL Server |

### Workflow Steps

#### Step 1: Create Azure SQL Infrastructure
1. Run `az config set extension.use_dynamic_install=yes_without_prompt`
2. Check if Entra group `itsm-sql-admins` already exists (`az ad group show --group itsm-sql-admins`)
   - If not, create it: `az ad group create --display-name itsm-sql-admins --mail-nickname itsm-sql-admins`
   - Ensure the current user is a member; add if not: `az ad group member add --group itsm-sql-admins --member-id <USER_OID>`
3. Create resource group (`cdw-itsmtesting-20260207-v02` in `westus2`)
4. Create Azure SQL Server (`itsm20260207v02`) with:
   - Entra-only authentication (`--enable-ad-only-auth`)
   - System Assigned Managed Identity (`--assign-identity --identity-type SystemAssigned`)
   - The Entra group as AD admin (`--external-admin-principal-type Group`)
5. Create empty database (`ITSM`) with:
   - General Purpose Serverless SKU (`--edition GeneralPurpose --compute-model Serverless --family Gen5 --capacity 1`)
   - Auto-pause after 60 min (`--auto-pause-delay 60`)
   - Local backup redundancy (`--backup-storage-redundancy Local`)
6. Configure firewall rules:
   - Allow current client IP (`curl -s https://ifconfig.me`)
   - Allow Azure services (`0.0.0.0` to `0.0.0.0`)

#### Step 2: Import BACPAC via ACI + SqlPackage
This is the proven approach. Do NOT try to run SqlPackage locally — it will likely fail due to token expiry over slow networks.

1. **Create ephemeral resources:**
   - Create a user-assigned managed identity (`bacpac-import-identity`)
   - Add the MI to the `itsm-sql-admins` Entra group (no need to swap the SQL admin)
   - Wait 60 seconds for the group membership to propagate
   - Create an ACR (`itsm20260207v02acr`, Basic SKU, admin enabled)

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
   - Remove the MI from the `itsm-sql-admins` Entra group
   - Delete ACI: `az container delete --name bacpac-import`
   - Delete ACR: `az acr delete --name itsm20260207v02acr`
   - Delete managed identity: `az identity delete --name bacpac-import-identity`

#### Step 3: Mirror to Microsoft Fabric
**Prerequisites (automated in Step 1):**
- SQL Server System Assigned Managed Identity (SAMI) must be enabled and be the primary identity
- "Allow Azure services" firewall rule must be configured

**Fabric capacity prerequisite:**
- Fabric mirroring requires a **Fabric capacity** (F SKU). Premium Per User (PPU/PP3) does NOT work.
- Create an F2 capacity (smallest, ~$262/month) in the same resource group:
  ```bash
  az fabric capacity create \
    --resource-group cdw-itsmtesting-20260207-v02 \
    --capacity-name itsmfabric20260207 \
    --location westus2 \
    --sku "{name:F2,tier:Fabric}" \
    --administration "{members:[youruser@domain.com]}"
  ```
  Note: This will auto-register the `Microsoft.Fabric` resource provider if needed (~2-3 min).

**Database setup (run once after import):**
1. Create a contained database user for your Entra identity with mirroring permissions:
   ```sql
   CREATE USER [youruser@domain.com] FROM EXTERNAL PROVIDER;
   GRANT SELECT, ALTER ANY EXTERNAL MIRROR, VIEW DATABASE PERFORMANCE STATE, VIEW DATABASE SECURITY STATE TO [youruser@domain.com];
   ```

**Configure mirroring via Fabric REST API:**
1. **Create a Fabric workspace** (if not already created):
   ```bash
   TOKEN=$(az account get-access-token --resource https://api.fabric.microsoft.com --query accessToken -o tsv)
   curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
     -d '{"displayName": "My ITSM Workspace"}' \
     "https://api.fabric.microsoft.com/v1/workspaces"
   ```
   Record the workspace ID from the response.

2. **Assign workspace to the F2 capacity:**
   ```bash
   # Get the F2 capacity ID
   curl -s -H "Authorization: Bearer $TOKEN" "https://api.fabric.microsoft.com/v1/capacities"
   # Assign it
   curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
     -d '{"capacityId": "<F2_CAPACITY_ID>"}' \
     "https://api.fabric.microsoft.com/v1/workspaces/<WORKSPACE_ID>/assignToCapacity"
   ```

3. **Enable workspace identity** (required for WorkspaceIdentity auth):
   ```bash
   curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Length: 0" \
     "https://api.fabric.microsoft.com/v1/workspaces/<WORKSPACE_ID>/provisionIdentity"
   ```
   Wait ~20 seconds for provisioning. The workspace identity name matches the workspace name.

4. **Create a Fabric connection** to the Azure SQL Database:
   ```bash
   curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
     -d '{
       "connectivityType": "ShareableCloud",
       "displayName": "ITSM-AzureSQL-Connection",
       "connectionDetails": {
         "type": "SQL", "creationMethod": "SQL",
         "parameters": [
           {"dataType": "Text", "name": "server", "value": "itsm20260207v02.database.windows.net"},
           {"dataType": "Text", "name": "database", "value": "ITSM"}
         ]
       },
       "privacyLevel": "Organizational",
       "credentialDetails": {
         "singleSignOnType": "None", "connectionEncryption": "NotEncrypted",
         "skipTestConnection": false,
         "credentials": {"credentialType": "WorkspaceIdentity"}
       }
     }' "https://api.fabric.microsoft.com/v1/connections"
   ```
   Record the connection ID from the response.

5. **Create a SQL user for the workspace identity** (the workspace identity name matches the workspace name):
   ```sql
   CREATE USER [My ITSM Workspace] FROM EXTERNAL PROVIDER;
   GRANT SELECT, ALTER ANY EXTERNAL MIRROR, VIEW DATABASE PERFORMANCE STATE, VIEW DATABASE SECURITY STATE TO [My ITSM Workspace];
   ```

6. **Create the mirrored database** using the connection ID:
   ```bash
   # Build the mirroring.json payload (mirrors all tables)
   PAYLOAD=$(echo -n '{
     "properties": {
       "source": {
         "type": "AzureSqlDatabase",
         "typeProperties": {"connection": "<CONNECTION_ID>", "database": "ITSM"}
       },
       "target": {
         "type": "MountedRelationalDatabase",
         "typeProperties": {"defaultSchema": "dbo", "format": "Delta"}
       }
     }
   }' | base64 -w0)

   curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
     -d "{
       \"displayName\": \"ITSM Mirrored Database\",
       \"description\": \"Mirror of ITSM Azure SQL Database\",
       \"definition\": {
         \"parts\": [{\"path\": \"mirroring.json\", \"payload\": \"$PAYLOAD\", \"payloadType\": \"InlineBase64\"}]
       }
     }" "https://api.fabric.microsoft.com/v1/workspaces/<WORKSPACE_ID>/mirroredDatabases"
   ```
   Record the mirrored database ID from the response.

7. **Grant the SQL Server SAMI access** to the mirrored database:
   - Get the SQL Server SAMI object ID: `az sql server show --name itsm20260207v02 --resource-group cdw-itsmtesting-20260207-v02 --query identity.principalId -o tsv`
   - Add the SAMI as a workspace contributor via the Fabric API:
     ```bash
     curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
       -d '{"principal": {"id": "<SAMI_OID>", "type": "ServicePrincipal"}, "role": "Contributor"}' \
       "https://api.fabric.microsoft.com/v1/workspaces/<WORKSPACE_ID>/roleAssignments"
     ```

8. **Start mirroring:**
   ```bash
   curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Length: 0" \
     "https://api.fabric.microsoft.com/v1/workspaces/<WORKSPACE_ID>/mirroredDatabases/<MIRRORED_DB_ID>/startMirroring"
   ```

9. **Monitor mirroring status:**
   ```bash
   # Overall status (expect "Running")
   curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Length: 0" \
     "https://api.fabric.microsoft.com/v1/workspaces/<WORKSPACE_ID>/mirroredDatabases/<MIRRORED_DB_ID>/getMirroringStatus"

   # Per-table status
   curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Length: 0" \
     "https://api.fabric.microsoft.com/v1/workspaces/<WORKSPACE_ID>/mirroredDatabases/<MIRRORED_DB_ID>/getTablesMirroringStatus"
   ```

**Alternative: Configure mirroring via Fabric portal:**
1. Open [fabric.microsoft.com](https://fabric.microsoft.com) and navigate to your workspace
2. Select **Create** → **Mirrored Azure SQL Database** (under Data Warehouse section)
3. Under **New sources**, select **Azure SQL Database**
4. Enter: Server `itsm20260207v02.database.windows.net`, Database `ITSM`, Auth: Organization account
5. Select **Connect** → select tables → **Mirror database**

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

### Why an Entra ID group as SQL admin?
- SQL Server supports a single AD admin, which can be a user or a group.
- Using a group (`itsm-sql-admins`) allows both your user and the ACI managed identity to have admin access simultaneously.
- No need to swap the admin back and forth — just add/remove members from the group.
- The group persists across resource recreations, making the workflow more robust.

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
8. **Use an Entra ID group as SQL Server admin** — Create a security group (e.g., `itsm-sql-admins`), add your user to it, and set the group as the SQL admin. During import, add the ACI managed identity to the same group. This avoids swapping the admin back and forth.

### SqlPackage
9. **SqlPackage version must match bacpac schema** — SQL Server 2022 bacpac (Sql170) requires SqlPackage 170.x.
10. **Use the standalone Linux zip for containers** — Download from `https://aka.ms/sqlpackage-linux` rather than the .NET tool install.
11. **SqlPackage only reads local files** — No support for Azure Blob Storage URLs as source.

### General Azure
12. **Run `az config set extension.use_dynamic_install=yes_without_prompt`** — Prevents interactive prompts for CLI extension installs.
13. **Register resource providers before first use** — `Microsoft.ContainerRegistry` and `Microsoft.ContainerInstance` may not be registered on the subscription. Use `az provider register --namespace <ns> --wait`.
14. **Blob storage is unnecessary for this workflow** — Since SqlPackage can't use blob URLs and `az sql db import` can't use Entra-only auth, staging the bacpac in blob storage adds no value.

### Fabric Mirroring
15. **Fabric mirroring requires an F SKU capacity** — PPU (Premium Per User / PP3) does NOT work for Fabric workloads. Create an F2 capacity via `az fabric capacity create` (~$262/month). Trial capacity (F4/F64) also works if available on your tenant.
16. **The entire mirroring workflow can be done via REST API** — Create workspace, assign capacity, provision workspace identity, create connection, mirrored database, start mirroring, and monitor status all via `https://api.fabric.microsoft.com/v1/`. No portal required.
17. **Use `WorkspaceIdentity` credential type for connections** — OAuth2 credential type requires interactive browser auth; `WorkspaceIdentity` works from CLI/API without browser interaction. Must enable workspace identity first via `provisionIdentity` API.
18. **The workspace identity needs a SQL database user** — Create a contained user with the workspace name (e.g., `CREATE USER [My ITSM Workspace] FROM EXTERNAL PROVIDER`) and grant mirroring permissions. Without this, mirroring fails with `SqlAuthenticationFailure`.
19. **SQL Server SAMI needs Contributor role on the Fabric workspace** — After creating the mirrored database, grant the SQL Server's System Assigned MI the Contributor role on the workspace via the `roleAssignments` API. This allows the SAMI to write mirrored data.
20. **Fabric workspace and Entra group persist across teardowns** — Like the `itsm-sql-admins` Entra group, the Fabric workspace and connection can be kept when deleting Azure resource groups. The F2 capacity lives in the resource group and will be deleted with it.
21. **`Microsoft.Fabric` provider may need registration** — First `az fabric capacity create` auto-registers the provider, but it takes ~2-3 minutes.

### Success Markers

- [x] Resource group created
- [x] SQL Server created with Entra-only auth and system MI
- [x] Database created with serverless SKU
- [x] Firewall configured (client IP + Azure services)
- [x] BACPAC imported successfully via ACI
- [x] Ephemeral resources cleaned up (ACR, ACI, MI)
- [x] Fabric F2 capacity created and workspace assigned
- [x] Fabric mirroring configured and running (500 tables)
