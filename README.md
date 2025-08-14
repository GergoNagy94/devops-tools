# DevOps Tools - GitHub Actions

Custom GitHub Actions for DevOps automation and infrastructure management.

## Terraform Module Version Checker

This action helps identify and track the versions of Terraform/OpenTofu modules used in your `terragrunt.hcl` files (Terragrunt Units). It scans a directory and checks the current and latest available versions of your modules.

### Usage

```yaml
- uses: your-username/company-github-actions/terraform-version-checker@v1
  with:
    environment: 'units'          # Required: directory to scan
    outdated-only: false          # Optional: show only outdated modules
```

### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `environment` | Environment directory to scan for `terragrunt.hcl` files | Yes | `units` |
| `outdated-only` | Show only outdated modules in the report | No | `false` |

### Outputs

| Output | Description |
|--------|-------------|
| `report-file` | Path to the generated YAML report file |
| `outdated-count` | Number of outdated modules found (useful for conditions) |

### Generated Reports

The action generates YAML files in the scanned directory:

- **Full report**: `terraform-versions-<directory>.yml`
- **Outdated only**: `terraform-versions-<directory>-outdated.yml` (when `outdated-only: true`)

#### Report Structure

```yaml
# Generated: 2025-08-14 10:30:15 UTC
# Environment: units
# Report Type: Full Report

terraform_modules:
  - unit_name: "vpc"                    # Terragrunt unit name (directory name)
    module_name: "terraform-aws-modules/vpc/aws"  # Official module name
    module_type: "git"                  # Source type: git, local, or unknown
    current_version: "v5.1.2"           # Version in terragrunt.hcl
    latest_version: "v5.8.1"            # Latest available version from GitHub
    status: "outdated"                  # up-to-date or outdated
    needs_update: true                  # Boolean: true if module needs update
    
  - unit_name: "security-group"
    module_name: "terraform-aws-modules/security-group/aws"
    module_type: "git"
    current_version: "v5.1.0" 
    latest_version: "v5.1.0"
    status: "up-to-date"
    needs_update: false
```

### How It Works

1. **Parse Configuration**: Determines target directory and report mode
2. **Dependency Check**: Verifies required tools (`curl`, `jq`) are available
3. **File Discovery**: Recursively finds all `terragrunt.hcl` files in the target directory
4. **Module Analysis**: For each file:
   - Extracts the `source` line from the `terraform{}` block
   - Determines module type: `git`, `local`, or `unknown`
   - For Git modules: parses GitHub URL and current version reference
   - Uses GitHub API to fetch the latest available version
   - Compares current vs latest versions
5. **Report Generation**: Creates structured YAML output with version comparison results

### Directory Structure

The action expects this structure:
```
your-repo/
├── .github/workflows/
└── units/                    # or your custom environment directory
    ├── vpc/
    │   └── terragrunt.hcl
    ├── security-groups/
    │   └── terragrunt.hcl
    └── ...
```

### Example Workflow

```yaml
name: Check Terraform Module Versions

on:
  schedule:
    - cron: '0 9 * * 1'  # Weekly on Monday at 9 AM
  workflow_dispatch:

jobs:
  check-versions:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Check all modules
        id: check-all
        uses: ./terraform-version-checker
        with:
          environment: 'units'
          outdated-only: false
      
      - name: Check outdated only
        id: check-outdated
        uses: ./terraform-version-checker
        with:
          environment: 'units'
          outdated-only: true
      
      - name: Report results
        run: |
          echo "Full report: ${{ steps.check-all.outputs.report-file }}"
          echo "Outdated modules: ${{ steps.check-outdated.outputs.outdated-count }}"
          
          if [[ "${{ steps.check-outdated.outputs.outdated-count }}" -gt "0" ]]; then
            echo "::warning::Found ${{ steps.check-outdated.outputs.outdated-count }} outdated modules"
            cat "${{ steps.check-outdated.outputs.report-file }}"
          fi

      - name: Upload reports
        uses: actions/upload-artifact@v4
        with:
          name: terraform-version-reports
          path: |
            ${{ steps.check-all.outputs.report-file }}
            ${{ steps.check-outdated.outputs.report-file }}
```

### Supported Module Sources

- **Git modules**: `git::https://github.com/terraform-aws-modules/vpc.git?ref=v5.1.2`
- **Local modules**: `../modules/custom-vpc`  
- **Unknown sources**: Any other source format (marked as `unknown`)

Only Git modules hosted on GitHub will have version checking via the GitHub API.

## AWS Execution Role Setup

Automates IAM role creation for Terragrunt deployments. GitHub Actions equivalent of `./make.sh dev.json`.

### Usage

**Create/update IAM role:**
```yaml
- uses: GergoNagy94/devops-tools/aws-execution-role@v1
  with:
    config-file: 'dev.json'
```

**Generate configuration template:**
```yaml
- uses: GergoNagy94/devops-tools/aws-execution-role@v1
  with:
    generate-template: true
    account-id: '567749996660'
    environment-name: 'dev'
```

### Configuration File (dev.json)

```json
{
  "account_id": "567749996660",
  "name": "dev",
  "policies": {
    "trust": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "AWS": ["arn:aws:iam::${account_id}:root"],
            "Service": "vpc-flow-logs.amazonaws.com"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    },
    "inline": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": ["acm:RequestCertificate", "acm:AddTagsToCertificate", "acm:DescribeCertificate", "acm:ListTagsForCertificate", "acm:DeleteCertificate"],
          "Resource": ["arn:aws:acm:eu-west-2:${account_id}:certificate/*", "arn:aws:acm:eu-north-1:${account_id}:certificate/*"]
        },
        {
          "Effect": "Allow", 
          "Action": ["ecr:CreateRepository", "ecr:DeleteRepository", "ecr:TagResource", "ecr:DescribeRepositories", "ecr:ListTagsForResource", "ecr:SetRepositoryPolicy", "ecr:PutLifecyclePolicy", "ecr:GetRepositoryPolicy", "ecr:GetLifecyclePolicy", "ecr:DeleteLifecyclePolicy", "ecr:DeleteRepositoryPolicy"],
          "Resource": ["arn:aws:ecr:eu-west-2:${account_id}:repository/*", "arn:aws:ecr:eu-north-1:${account_id}:repository/*"]
        },
        {
          "Effect": "Allow",
          "Action": ["eks:*"],
          "Resource": ["arn:aws:eks:eu-west-2:${account_id}:*", "arn:aws:eks:eu-north-1:${account_id}:*"]
        },
        {
          "Effect": "Allow",
          "Action": ["iam:CreateRole", "iam:DeleteRole", "iam:PassRole", "iam:GetRole", "iam:ListRolePolicies", "iam:ListAttachedRolePolicies", "iam:TagRole", "iam:UntagRole", "iam:CreateServiceLinkedRole", "iam:AttachRolePolicy", "iam:ListInstanceProfilesForRole", "iam:DetachRolePolicy", "iam:PassRole", "iam:PutRolePolicy", "iam:GetRolePolicy", "iam:CreateOpenIDConnectProvider", "iam:DeleteRolePolicy", "iam:TagOpenIDConnectProvider", "iam:GetOpenIDConnectProvider", "iam:DeleteOpenIDConnectProvider", "iam:UpdateAssumeRolePolicy", "iam:CreateInstanceProfile", "iam:TagInstanceProfile", "iam:GetInstanceProfile", "iam:DeleteInstanceProfile", "iam:RemoveRoleFromInstanceProfile", "iam:AddRoleToInstanceProfile", "iam:DeleteServiceLinkedRole", "iam:GetServiceLinkedRoleDeletionStatus"],
          "Resource": ["arn:aws:iam::${account_id}:role/*", "arn:aws:iam::${account_id}:oidc-provider/*", "arn:aws:iam::${account_id}:instance-profile/*"]
        },
        {
          "Effect": "Allow",
          "Action": ["s3:*"],
          "Resource": "*"
        }
      ]
    },
    "managed": [
      "arn:aws:iam::aws:policy/AmazonVPCFullAccess",
      "arn:aws:iam::aws:policy/AmazonEC2FullAccess",
      "arn:aws:iam::aws:policy/AmazonElastiCacheFullAccess",
      "arn:aws:iam::aws:policy/CloudFrontFullAccess",
      "arn:aws:iam::aws:policy/IAMFullAccess",
      "arn:aws:iam::aws:policy/AmazonRDSFullAccess",
      "arn:aws:iam::aws:policy/AmazonRoute53FullAccess",
      "arn:aws:iam::aws:policy/AmazonSSMFullAccess",
      "arn:aws:iam::aws:policy/AmazonMQFullAccess",
      "arn:aws:iam::aws:policy/AmazonOpenSearchServiceFullAccess"
    ]
  }
}
```

### How It Works

Like the original `make.sh` script:
1. Reads the JSON config file
2. Replaces `${account_id}` placeholders
3. Creates role if it doesn't exist
4. Attaches managed policies (non-destructive)
5. Updates inline policy

### Example Workflow

```yaml
name: Setup AWS Execution Role

on:
  workflow_dispatch:
    inputs:
      config-file:
        description: 'Path to config file (e.g., dev.json)'
        required: true
        default: 'dev.json'

jobs:
  setup-role:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Setup execution role
        uses: GergoNagy94/devops-tools/aws-execution-role@v1
        with:
          config-file: ${{ github.event.inputs.config-file }}
```

## Development

These actions are built as composite actions and can be tested locally or in GitHub Actions workflows.