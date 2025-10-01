# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

Project overview
- Purpose: Minimal Terraform configuration that provisions a public EC2 instance running Apache on port 8080, wired to a Terraform Cloud workspace and automated via GitHub Actions.
- Stack: Terraform (>= 1.1), Providers: aws (v4.52.0), random (v3.4.3). CI/CD: GitHub Actions + Terraform Cloud (TFC) via the official tfc-workflows actions.

High-level architecture
- Terraform Cloud integration
  - main.tf includes a cloud block pointing to an organization and a workspace. This enables remote operations/state in TFC.
  - GitHub Actions do not call terraform locally. They upload configuration to TFC and create runs via the TFC Workflows GitHub Actions.
- Providers and resources
  - Providers: aws (region us-east-1) and random.
  - Data: aws_ami.ubuntu selects the latest Ubuntu 20.04 AMI from Canonical (owner 675105128665) using filters.
  - Resources:
    - random_pet.sg: Generates a unique suffix for the security group name.
    - aws_security_group.web-sg: Allows inbound TCP 8080 from 0.0.0.0/0; egress is open.
    - aws_instance.web: t2.micro instance using the AMI above and the security group; user_data installs Apache, switches it to port 8080, and writes a “Hello World” index.html.
  - Output: web-address exposes the public DNS with :8080 appended.
- CI/CD (GitHub Actions)
  - .github/workflows/terraform-plan.yml (trigger: pull_request)
    - Uploads configuration to TFC as a speculative run, then comments plan summary on the PR.
    - Skips if the repo is the upstream template (hashicorp-education/learn-terraform-github-actions).
  - .github/workflows/terraform-apply.yml (trigger: push to main)
    - Uploads configuration, creates a run, and applies via TFC when confirmable.
    - Skips in the upstream template repo.
  - .github/workflows/your-fork.yml
    - Auto-closes PRs opened against the upstream template to nudge contributors to fork first.

Common development commands (PowerShell)
- Format and validate
  ```powershell path=null start=null
  terraform fmt -check -recursive
  terraform fmt -recursive   # to fix formatting
  terraform validate
  ```
- Initialize (local CLI)
  - If using Terraform Cloud from the CLI, authenticate first:
  ```powershell path=null start=null
  terraform login
  ```
  - Then initialize the working directory:
  ```powershell path=null start=null
  terraform init
  ```
- Plan and apply (CLI-driven runs)
  - Ensure the cloud block in main.tf points at your org/workspace, or set environment variables to match your intended workspace in TFC before running CLI commands.
  ```powershell path=null start=null
  $env:TF_CLOUD_ORGANIZATION = "<your_org>"
  $env:TF_WORKSPACE = "<your_workspace>"
  terraform plan
  terraform apply
  ```
- Destroy (teardown)
  ```powershell path=null start=null
  terraform destroy
  ```
Notes
- This repo contains no unit tests; there is no single-test target to run.
- Provider credentials:
  - For local CLI use, Terraform relies on the AWS SDK default credential chain (e.g., environment variables or a credentials file).
  - For Terraform Cloud runs (used by CI), configure AWS credentials as Terraform Cloud Workspace variables (e.g., AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, and region as needed).

CI/CD configuration essentials
- GitHub repository secrets/variables
  - TF_API_TOKEN must be configured as a repository secret so workflows can communicate with TFC:
    - In GitHub: Settings > Secrets and variables > Actions > New repository secret named TF_API_TOKEN with a Terraform Cloud user/team token.
  - Workflow environment variables are set within the YAML files:
    - TF_CLOUD_ORGANIZATION
    - TF_WORKSPACE
    - CONFIG_DIRECTORY (usually ./)
- Synchronizing workspace settings
  - main.tf currently sets a cloud.organization placeholder and a specific workspace name. Align these with the values used by the GitHub workflows to avoid confusion between CLI-driven and workflow-driven runs.
- Pull request experience
  - The plan workflow posts a concise summary to PRs and links to the full TFC run.

First-time setup checklist
1) Terraform Cloud
   - Create or identify an organization in TFC.
   - Create a workspace matching the value used by workflows (TF_WORKSPACE) or update the workflow to your workspace’s name.
   - Add AWS credentials as Workspace variables for remote applies.
2) Repository
   - Set the TF_API_TOKEN GitHub secret to a valid TFC token.
   - Optionally update .github/workflows/* to your org/workspace names.
3) Code
   - Update the cloud block in main.tf to your TFC organization and workspace if you plan to run terraform from the CLI.

Reference
- README.md notes that this repository mirrors the HashiCorp tutorial repo for automating Terraform with GitHub Actions.
