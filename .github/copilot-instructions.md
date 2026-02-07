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

*(Replace this section with your project-specific details)*

### What It Does
*(Describe the purpose and behavior of the application)*

### Technology Stack
*(List frameworks, SDKs, and cloud services used)*

### Azure Resources Required
*(List Azure resources needed, with links to setup docs)*

### Configuration Values
*(Table of configuration keys and descriptions — no actual values)*

### Build & Run

```bash
# Example:
# dotnet restore && dotnet build && dotnet run
```

---

## Lessons Learned & Best Practices

*(Update this section as you encounter issues, discover solutions, or identify patterns worth remembering)*

### Common Pitfalls
*(Document issues encountered and their resolutions)*

### Success Markers
*(Checklist of milestones to track progress)*
