---
inclusion: auto
---

# Terraform Conventions — Platform Engineering Standards

When generating or reviewing Terraform code, always follow these conventions.

## Module Source

- All infrastructure modules live in private GitHub repos named `myorg-terraform-aws-<service>`.
- Always use git source: `git::https://github.com/<org>/myorg-terraform-aws-<service>.git?ref=<version>`
- Pin to a specific tag — never use unversioned `main` in production.
- Always fetch the latest git tag for a module using `list_module_versions` and use that tag as the `ref` value in the module source.

## Required Tags

Every resource must include these tags (enforced via `default_tags` in the provider block):

| Tag         | Description                        |
|-------------|------------------------------------|
| Environment | `dev`, `staging`, or `prod`        |
| Team        | Team that owns the resource        |
| CostCenter  | Cost center for billing allocation |
| ManagedBy   | Always set to `terraform`          |

## Backend Configuration

- Backend/state management is handled by the CI/CD pipeline — do NOT generate `backend.tf`.
- Never include backend blocks in scaffolded Terraform configs.

## Provider Requirements

- Terraform version: `>= 1.14`
- AWS provider version: `>= 5.0`
- Always use `required_providers` block.

## File Structure

Every Terraform project must have:

```
providers.tf    — terraform block, required_providers, provider config
variables.tf    — input variables (environment, team, cost_center + module-specific)
terraform.tfvars — actual values for all variables passed in module calls
data.tf         — all data sources (aws_iam_policy_document, aws_ami, etc.)
main.tf         — module calls and resources only (NO data sources)
outputs.tf      — output values from modules
```

## Variable Values (`terraform.tfvars`)

- All variables passed in module calls must be defined in `terraform.tfvars`.
- Never hardcode values directly in `main.tf` module blocks — reference variables instead, and set their values in `terraform.tfvars`.
- This keeps configuration DRY and makes it easy to change values per environment without touching module code.

**Important:** `data` resources (e.g., `data "aws_iam_policy_document"`, `data "aws_ami"`) must NEVER be placed in `main.tf`. They belong in `data.tf`.

## IAM Policy Conventions

- Always use **customer managed policies** (`aws_iam_policy` + `aws_iam_role_policy_attachment`) when creating IAM policies.
- Do NOT use inline policies (`aws_iam_role_policy` or the `inline_policies` argument on IAM modules). Inline policies are harder to audit, reuse, and manage at scale.
- Define policy documents in `data.tf` using `data "aws_iam_policy_document"`, then reference them in a managed policy resource.

## Naming Conventions

- Resource names: `<team>-<service>-<environment>` (e.g., `platform-vpc-dev`)
- Variable names: `snake_case`
- Output names: `snake_case`, prefixed with module context when needed

## Documentation

- Every infrastructure project must include a `README.md` at the root of the project directory.
- The README should cover: purpose of the infrastructure, modules used, input variables, outputs, and any prerequisites or usage instructions.

## Workflow

When a user asks to create infrastructure:
1. Use `search_modules` to find the relevant module
2. Use `list_module_versions` to fetch the latest git tag for the module
3. Use `get_module` to inspect variables and outputs
4. Use `scaffold_terraform` to generate the full config, using the latest tag as the `ref`
5. Write the generated files to the workspace
6. Remind the user to run `terraform init` and `terraform plan`
