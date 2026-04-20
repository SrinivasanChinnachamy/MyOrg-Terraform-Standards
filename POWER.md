---
name: "myorg-terraform-modules"
displayName: "Centralized Terraform Modules & IaC Standards"
description: "Discover, inspect, and scaffold Terraform configurations from private GitHub-hosted modules following the myorg-terraform-aws-* naming convention. Enforces platform engineering standards including required tags, provider pinning, and file structure conventions."
keywords: ["terraform", "myorg-terraform", "infrastructure", "scaffold", "modules"]
author: "Srinivasan Chinnachamy"
---

# Organisation Terraform Modules

## Overview

This power provides an end-to-end workflow for building AWS infrastructure using private Terraform modules hosted in GitHub. It connects to the `myorg-terraform-mcp-server`, which discovers repositories matching the `myorg-terraform-aws-*` naming convention and generates complete, standards-compliant Terraform configurations.

The server enforces platform engineering conventions — required tags (Environment, Team, CostCenter, ManagedBy), provider version pinning (Terraform >= 1.14, AWS >= 5.0), and a strict file structure — so every generated config is production-ready from the start.

This power also includes agent hooks for automated compliance checks and a steering document that codifies your team's Terraform conventions, ensuring the AI agent always follows your standards.

## Onboarding

### Prerequisites

- Python 3.10+ with `uv` / `uvx` installed ([installation guide](https://docs.astral.sh/uv/getting-started/installation/))
- A GitHub personal access token with `repo` scope (for accessing private module repos)
- Terraform CLI >= 1.14 installed locally
- Access to the GitHub organization hosting the `myorg-terraform-aws-*` module repos

### Installation

The MCP server is installed automatically via `uvx` when the power is activated. No manual package installation is needed.

### Configuration

After installing this power, replace the placeholder values in `mcp.json`:

## MCP Config Placeholders

- **`YOUR_GITHUB_TOKEN`**: A GitHub personal access token with `repo` scope for accessing private repositories.
  - **How to get it:**
    1. Go to https://github.com/settings/tokens
    2. Click "Generate new token (classic)"
    3. Select the `repo` scope
    4. Copy the generated token
    5. Set it as an environment variable: `export GITHUB_TOKEN=ghp_your_token_here`

- **`YOUR_GITHUB_ORG`**: The GitHub organization or username that owns the Terraform module repositories.
  - **How to set it:** Use the org or username where your `myorg-terraform-aws-*` repos live (e.g., `SrinivasanChinnachamy`)

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `GITHUB_TOKEN` | GitHub personal access token with `repo` scope | Yes |
| `GITHUB_ORG` | GitHub org or user owning the module repos | Yes |
| `MODULE_PREFIX` | Repo naming prefix (default: `myorg-terraform-aws-`) | No |
| `TF_MIN_VERSION` | Minimum Terraform version (default: `1.14`) | No |

## Common Workflows

### Workflow 1: Discover Available Modules

Search for all available Terraform modules or filter by AWS service name.

**Steps:**
1. Use `search_modules` with no keyword to list all modules
2. Or filter by service: `search_modules(keyword="ec2")` or `search_modules(keyword="s3")`

**Example:**
```
search_modules(keyword="ec2")
→ Returns: module name, service, description, GitHub URL, last updated
```

### Workflow 2: Inspect a Module

Get full details about a module — its variables, outputs, and README.

**Steps:**
1. Use `get_module(service_name="ec2")` to fetch module details
2. Review the variables to understand required inputs
3. Review outputs to know what the module exposes

**Example:**
```
get_module(service_name="s3", ref="v1.0.0")
→ Returns: source URL, variables with types/defaults, outputs, README content
```

### Workflow 3: Scaffold a Full Terraform Configuration

Generate a complete, standards-compliant Terraform project from a module.

**Steps:**
1. Search for the module: `search_modules(keyword="ec2")`
2. Get the latest version: `list_module_versions(service_name="ec2")`
3. Inspect the module: `get_module(service_name="ec2", ref="v1.0.0")`
4. Scaffold the config: `scaffold_terraform(service_name="ec2", ref="v1.0.0", environment="dev", team="platform", cost_center="engineering")`
5. Write the generated files to your workspace
6. Run `terraform init` and `terraform plan`

**Example:**
```
scaffold_terraform(
  service_name="ec2",
  module_variables='{"instance_type": "t3.medium"}',
  environment="prod",
  team="backend",
  cost_center="product",
  ref="v1.0.0"
)
→ Returns: providers.tf, variables.tf, main.tf, outputs.tf — all ready to use
```

**Generated file structure:**
```
providers.tf    — terraform block, required_providers, provider with default_tags
variables.tf    — environment, team, cost_center + module-specific variables
main.tf         — module call with variable references
outputs.tf      — module output passthrough
```

### Workflow 4: Check Module Versions

List all available git tags for a module to find the latest release.

**Steps:**
1. Use `list_module_versions(service_name="ec2")`
2. Pick the latest tag for your module source `ref`

**Example:**
```
list_module_versions(service_name="s3")
→ Returns: list of tags with name and commit SHA
```

## Platform Engineering Standards

The following conventions are enforced by the MCP server, steering document, and agent hooks.

### Required Tags

Every resource must include these tags via `default_tags` in the provider block:

| Tag | Description |
|-----|-------------|
| `Environment` | `dev`, `staging`, or `prod` |
| `Team` | Team that owns the resource |
| `CostCenter` | Cost center for billing allocation |
| `ManagedBy` | Always set to `terraform` |

### File Structure

Every Terraform project must follow this layout:

```
providers.tf     — terraform block, required_providers, provider config
variables.tf     — input variables (environment, team, cost_center + module-specific)
terraform.tfvars — actual values for all variables
data.tf          — all data sources (aws_iam_policy_document, aws_ami, etc.)
main.tf          — module calls and resources only (NO data sources)
outputs.tf       — output values from modules
```

### Provider Requirements

- Terraform version: `>= 1.14`
- AWS provider version: `>= 5.0`
- Always use `required_providers` block

### Module Source Convention

- All modules live in private GitHub repos named `myorg-terraform-aws-<service>`
- Always use git source: `git::https://github.com/<org>/myorg-terraform-aws-<service>.git?ref=<version>`
- Pin to a specific tag — never use unversioned `main` in production
- Always fetch the latest tag using `list_module_versions` before scaffolding

### IAM Policy Conventions

- Always use customer managed policies (`aws_iam_policy` + `aws_iam_role_policy_attachment`)
- Never use inline policies (`aws_iam_role_policy`)
- Define policy documents in `data.tf` using `data "aws_iam_policy_document"`

### Backend Configuration

- Backend/state management is handled by the CI/CD pipeline
- Never include `backend.tf` or backend blocks in scaffolded configs

## Agent Hooks

This power includes three agent hooks that automate compliance checks:

### 1. Terraform Tag Convention Check (preToolUse)

Triggers before any file write operation. Verifies that `.tf` files include required tags (Environment, Team, CostCenter, ManagedBy) via `default_tags`, Terraform version >= 1.14, and S3 backend with encryption.

```json
{
  "name": "Terraform Tag Convention Check",
  "version": "1.0.0",
  "when": {
    "type": "preToolUse",
    "toolTypes": ["write"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "Before writing this .tf file, verify it includes the required tags (Environment, Team, CostCenter, ManagedBy) either directly or via default_tags in the provider block. Also verify the terraform version constraint is >= 1.14 and the backend uses S3 with encrypt = true. If any convention is missing, fix it before writing."
  }
}
```

### 2. Review Terraform Against Standards (fileEdited)

Triggers when steering files or skills are updated. Reviews all Terraform files in `infra-build/` to ensure they comply with the latest standards.

```json
{
  "name": "Review Terraform Against Standards",
  "version": "1.0.0",
  "when": {
    "type": "fileEdited",
    "patterns": [".kiro/steering/*.md", ".kiro/skills/*"]
  },
  "then": {
    "type": "askAgent",
    "prompt": "A steering file or skill in the workspace was just updated. Read the latest version of all steering files in .kiro/steering/ and skills in .kiro/skills/ (if they exist), then review every Terraform file in the infra-build/ folder to ensure they meet the latest standards and conventions. Report any deviations and suggest or apply fixes."
  }
}
```

### 3. Terraform Validate on Save (fileEdited)

Triggers when any `.tf` file is saved. Runs `terraform fmt -check -diff` to catch formatting issues immediately.

```json
{
  "name": "Terraform Validate on Save",
  "version": "1.0.0",
  "when": {
    "type": "fileEdited",
    "patterns": ["*.tf"]
  },
  "then": {
    "type": "runCommand",
    "command": "terraform fmt -check -diff . 2>&1 || true"
  }
}
```

## Steering Document

This power includes a steering document (`terraform-conventions.md`) that is auto-included in every agent interaction. It codifies the full set of platform engineering standards:

- Module source conventions and version pinning
- Required tags and provider configuration
- File structure rules (data sources in `data.tf`, not `main.tf`)
- IAM policy conventions (managed policies only, no inline)
- Variable management (`terraform.tfvars` for all values)
- Naming conventions (`<team>-<service>-<environment>`)
- The recommended workflow: search → version check → inspect → scaffold → write → init/plan

The steering document ensures the agent follows your team's standards in every interaction, even without explicit reminders.

## Troubleshooting

### MCP Server Connection Issues

**Problem:** Server won't start or tools aren't available
**Solutions:**
1. Verify `uvx` is installed: `uvx --version`
2. Check that `GITHUB_TOKEN` is set and valid
3. Test the token: `curl -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/user`
4. Restart Kiro and check the MCP Server panel for errors

### No Modules Found

**Problem:** `search_modules` returns empty results
**Solutions:**
1. Verify `GITHUB_ORG` is set correctly
2. Check that repos follow the `myorg-terraform-aws-*` naming convention
3. Ensure the GitHub token has access to the organization's repos
4. Try searching without a keyword to list all modules

### Scaffold Missing Variables

**Problem:** Generated config is missing expected variables
**Solutions:**
1. Use `get_module` first to inspect the module's variables
2. Pass overrides via `module_variables` as a JSON string
3. Variables without defaults will be added to `variables.tf` automatically

### Permission Denied on GitHub API

**Problem:** 403 or 401 errors from GitHub
**Solutions:**
1. Regenerate your GitHub token with `repo` scope
2. Verify the token hasn't expired
3. Check org membership and repo access permissions

## Best Practices

- Always use `list_module_versions` to get the latest tag before scaffolding — never reference `main` in production
- Review generated configs before running `terraform apply`
- Keep `terraform.tfvars` for environment-specific values — don't hardcode in `main.tf`
- Use the steering document to maintain consistency across team members
- Let the agent hooks catch compliance issues automatically during development

---

**Package:** `myorg-terraform-mcp-server`
**MCP Server:** myorg-terraform-mcp-server
**Source:** `git+https://github.com/SrinivasanChinnachamy/myorg-terraform-mcp-server.git`
