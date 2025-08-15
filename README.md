# DevOps Tools

Collection of GitHub Actions for infrastructure automation and AWS IAM management.

## Tools

### terraform-version-checker
Scans Terragrunt configurations to check if Terraform modules are up to date. Generates YAML reports showing current vs latest versions.

### aws-execution-policy-reader & aws-execution-policy-writer
These tools work together for IAM role management:
- **Reader**: Exports existing IAM role policies to JSON (backup/audit)
- **Writer**: Creates/updates IAM roles from JSON templates

The reader output can be used as input for the writer, enabling role migration between environments.

## Examples

### Terraform Version Checker
Workflow scans `infrastructure/live/dev` or `infrastructure/live/prod` directories, checks module versions against GitHub releases, and uploads YAML reports as artifacts.

### AWS Policy Reader  
Workflow reads IAM policies from multiple AWS accounts (dev/prod/management/monitoring), exports to JSON artifacts for backup or compliance auditing.

## Usage

See `.github/workflows/` directory for complete workflow configurations.