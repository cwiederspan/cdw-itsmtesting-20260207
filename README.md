# ITSM Testing Project

## Bootstrap Prompt

### Step 1: Setup the Instructions

---

# Goal

I need to develop a multi-step workflow that will:

- **Step 1** - Create a Azure SQL Database
- **Step 2** - Restore the ITSM.bacpac data to the database
- **Step 3** - Mirror the ITSM tables into Microsoft Fabric

I will need to execute these steps one-by-one instead of all at once. The restoration of the .bacpac file will take a long time, up to about 60 minutes, so do anything you can to make that process less fragile, faster, and asynchronous. And you will likely want to upload the .bacpac file to an Azure storage account first for better performance.

## Azure Resources Needed
- Resource Group
- Azure SQL Server
- Azure SQL Database
- Storage Account (for bacpac staging - required for async import)

## Requirements
* Keep it cheap, use General Purpose Serverless SKU
* Microsoft Entra authentication only
* Server has System Assigned Managed Identity enabled
* No HA, DR, or redundancy needed for the database or the backups
* Allow public endpoint, but with a firewall to limit by client IPs
* Never store secrets, passwords, credentials, or other sensitive data in this file

## Import Strategy
**Note:** `az sql db import` does NOT support Entra-only authentication. Use SqlPackage instead:
1. Upload .bacpac to Azure Blob Storage (optional, can import directly from local file)
2. Create empty database with serverless SKU
3. Use `sqlpackage /Action:Import` with `/AccessToken` for Entra auth
4. Import takes ~55 minutes for a 145MB bacpac file

## Available Tools
- Azure CLI - make sure you install the latest version before starting
- Azure MCP - prompt the user to set it up if its not already configured
- Use any other available tools as needed - you are not limited to this list

## Parameters
- Resource Group Name: `cdw-itsmtesting-20260207-v01`
- Location: `westus2`
- SQL Server Name: `itsm20260207v01` (must be globally unique)
- Database Name: `ITSM`
- Storage Account Name: `itsm20260207` (must be globally unique)
- Container Name: `bacpaccontainer`
- .bacpac File Path: `./ITSM.bacpac`

## Previous Lessons Learned
1. **Storage account key auth may be disabled by Azure Policy** - Use Entra auth with role assignments instead
2. **`az sql db import` does NOT work with Entra-only auth** - It requires SQL admin password; use SqlPackage instead
3. **SqlPackage version must match bacpac schema version** - SQL Server 2022 bacpac (Sql170) requires SqlPackage 170.x
4. **SqlPackage 170.2.70 requires .NET 8.0.20** - Use 170.0.94 if you have .NET 8.0.15
5. **Role assignments take ~60 seconds to propagate** - Wait before using newly assigned permissions
6. **User delegation SAS tokens work when key auth is disabled** - Use `--auth-mode login --as-user` for blob SAS
7. **Import of 145MB bacpac takes ~55 minutes** - Plan accordingly for larger databases

Use this information to update the .github/copilot-instructions.md file to support this goal. You will continue to update the .md file as you go with new information, lessons learned, problems or approaches to avoid, etc.

---

### Step 2: Launch GitHub Copilot

```bash

# It is good to do this right away
az login

# Launch GitHub Copilot CLI
copilot --yolo

# Log into GitHub
> /login

# Initialize the session with prompt from above
> /init

# Add plugins if desired

# Microsoft Work IQ
> /plugin marketplace add github/copilot-plugins
> /plugin install workiq@copilot-plugins 

# Microsoft Learn Docs
> /plugin marketplace add microsoftdocs/mcp
> /plugin install microsoft-docs@microsoft-docs-marketplace

```

### Step 3: Possible MCP Servers

- **Azure MCP** - `npx -y @azure/mcp@latest server start`
- **Microsoft Fabric** - `npx -y @microsoft/fabric-mcp@latest server start --mode all   `
- **Playwright** - `npx -y @playwright/mcp@latest`
- **Azure DevOps** - `npx -y @azure-devops/mcp your-org-name`
