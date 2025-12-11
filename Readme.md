
# terraform-aws-bootstrap

A Terraform module that bootstraps an AWS account with all resources required to manage remote Terraform state and (optionally) allow secure CI access via GitHub Actions.

The module provides:

* An S3 bucket for Terraform remote state
* A KMS key for state encryption
* Optional DynamoDB table for classic Terraform state locking
* Optional GitHub OIDC integration for secure AWS access from GitHub Actions
* Optional Datadog/Opsgenie resource permissions for teams that require them

This module is designed to be the **first Terraform workload** you run in a new AWS account.

---

## Overview

The module supports **two locking mechanisms**:

### 1. S3 native locking (recommended)

* Requires Terraform CLI **>= 1.10**
* No DynamoDB table is needed
* Enabled when:
  `enable_dynamodb_locking = false` (default)

### 2. DynamoDB-based locking (classic)

* Compatible with all Terraform versions
* Creates a DynamoDB table named `terraform`
* Enabled when:
  `enable_dynamodb_locking = true`

The user must select the correct backend configuration in the root module based on this setting.

---

## Requirements

### Terraform CLI

```
>= 1.10.0
```

Required for S3 native locking (`use_lockfile = true`).
DynamoDB locking works with older versions, but this module targets Terraform ≥ 1.10.

### AWS Provider

```
>= 5.0.0, < 6.0.0
```

This is the required **provider** version, not the Terraform CLI version.

---

## What the Module Creates

### Always created

* S3 bucket for Terraform state

  * Versioning enabled
  * KMS-encrypted
  * Public access blocked
  * Enforced TLS access
* KMS key and alias for encrypting Terraform state
* (Optional) GitHub OIDC provider and IAM role
* (Optional) Datadog/Opsgenie permission additions

### Conditionally created

#### DynamoDB table (for classic locking)

Created only when:

```
enable_dynamodb_locking = true
```

#### GitHub OIDC integration

Created only when:

```
enable_github_oidc = true
```

#### Datadog/Opsgenie permissions

Included only when:

```
enable_datadog_permissions = true
```

---


## Usage (Terraform Registry)

Once published, the module can be consumed like this:

```hcl
module "bootstrap" {
  source  = "Public-Cloud-Group/bootstrap/aws"
  version = "1.0.0"

  aws_account_id = var.aws_account_id
  region         = var.region
  oidc_repo      = "myorg/myrepo:*"
}
```

---

## Versioning

This module uses semantic versioning.
Always pin a version:

```hcl
version = "1.0.0"
```

---


## Inputs

| Name                              | Type   | Default                      | Description                                                                 |
| --------------------------------- | ------ | ---------------------------- | --------------------------------------------------------------------------- |
| `aws_account_id`                  | string | n/a                          | Target AWS account ID.                                                      |
| `region`                          | string | n/a                          | AWS region used for the provider and resources.                             |
| `oidc_repo`                       | string | n/a                          | GitHub Actions OIDC subject filter (e.g., `"org/repo:*"`).                  |
| `enable_dynamodb_locking`         | bool   | false                        | Whether to create a DynamoDB table for classic Terraform locking.           |
| `state_bucket_name`               | string | `""`                         | Custom S3 bucket name. If empty, a name is derived from the AWS account ID. |
| `kms_key_alias`                   | string | `"alias/tfstate"`            | Alias for the KMS key encrypting Terraform state.                           |
| `enable_github_oidc`              | bool   | true                         | Whether to create the GitHub OIDC provider and IAM role.                    |
| `enable_datadog_permissions`      | bool   | false                        | Whether to add Datadog/Opsgenie resource ARNs to the GitHub Actions role.   |
| `opsgenie_secret_name`            | string | `"opsgenie/api_key"`         | Secret name used if Datadog permissions are enabled.                        |
| `datadog_keys_secret_name`        | string | `"datadog/keys"`             | Secret name for Datadog keys.                                               |
| `datadog_integration_policy_name` | string | `"DatadogIntegrationPolicy"` | IAM policy name used by Datadog.                                            |
| `datadog_integration_role_name`   | string | `"DatadogIntegrationRole"`   | IAM role name used by Datadog.                                              |

---

## Outputs

| Name                       | Description                                                        |
| -------------------------- | ------------------------------------------------------------------ |
| `tf_state_bucket_name`     | Name of the S3 bucket storing Terraform state.                     |
| `tf_state_bucket_arn`      | ARN of the S3 state bucket.                                        |
| `tf_kms_key_arn`           | ARN of the KMS key used to encrypt the Terraform state.            |
| `tf_dynamodb_table_name`   | DynamoDB lock table name (null if not created).                    |
| `github_actions_role_arn`  | ARN of the IAM role created for GitHub Actions (null if disabled). |
| `github_oidc_provider_arn` | ARN of the GitHub OIDC provider (null if disabled).                |

---


## Using This Module in a Root Terraform Configuration

This module must be run in **two phases** because the remote backend (S3 and optional DynamoDB) does not exist until after the first `terraform apply`.

---

## Phase 1: Bootstrap Without a Backend

Your initial root structure:

```
live/
  └─ account-bootstrap/
       ├─ main.tf
       ├─ provider.tf
       ├─ variables.tf
       └─ (no backend.tf yet)
```

### main.tf

```hcl
module "bootstrap" {
  source  = "Public-Cloud-Group/bootstrap/aws"
  version = "1.0.0"

  aws_account_id          = var.aws_account_id
  region                  = var.region

  enable_github_oidc      = true
  oidc_repo               = "myorg/myrepo:*"

  enable_dynamodb_locking = false
  enable_datadog_permissions = false
}
```

### provider.tf

```hcl
provider "aws" {
  region              = var.region
  allowed_account_ids = [var.aws_account_id]
}
```

### **variables.tf**
```hcl
variable "region" {
  type    = string
  default = "eu-central-1"
}

variable "aws_account_id" {
  type    = string
  default = "123456789012"
}
```

### Run bootstrap locally

```
terraform init
terraform apply
```

This creates:

* S3 bucket for Terraform remote state
* KMS key
* DynamoDB table (if enabled)
* GitHub OIDC IAM resources (if enabled)
* Datadog/Opsgenie access expansions (if enabled)

---

## Phase 2: Configure Remote Backend After Resources Exist

Only after Phase 1 completes can you create `backend.tf`.

### backend.tf (S3 native locking)

```hcl
terraform {
  backend "s3" {
    bucket        = "terraform-state-<Account-ID>"
    key           = "state/bootstrap.tfstate"
    region        = "eu-central-1"
    use_lockfile  = true
  }
}
```
Notes:

* Requires Terraform CLI ≥ 1.10
* No DynamoDB table is involved

### backend.tf (DynamoDB locking)

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-<Account-ID>"
    key            = "state/bootstrap.tfstate"
    region         = "eu-central-1"
    dynamodb_table = "terraform"
  }
}
```
Notes:

* Supported by all Terraform CLI versions
* Requires the DynamoDB table to be created by the module

### Migrate state into the backend

```
terraform init -migrate-state
```

Now Terraform uses the S3 backend created by the module itself.

---

# Why This Two-Phase Process Is Required

Terraform requires the backend configuration **before** `terraform init`, but your backend resources are created **by** this module.

Therefore:

* First run locally (no backend) → create backend infrastructure
* Then define backend → migrate state

This is the standard pattern for Terraform bootstrapping modules.


---

## GitHub Actions Integration (optional)

When `enable_github_oidc = true`, the module:

1. Creates an AWS IAM OIDC provider referencing
   `https://token.actions.githubusercontent.com`
2. Generates a trust policy that allows GitHub Actions to assume an AWS role using OIDC
3. Restricts access using:

   * `aud = sts.amazonaws.com`
   * `sub` matched against your `oidc_repo` input (e.g., `"org/repo:*"`)

This avoids storing AWS credentials in GitHub and uses short-lived tokens instead.

Example use in GitHub Actions:

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.GITHUB_ACTIONS_ROLE_ARN }}
          aws-region: eu-central-1
```

---

## Datadog / Opsgenie Permissions (optional)

This is optional functionality.
It **does not create** Datadog resources.
Instead, it performs **lookups** (data sources) for resources that *already exist* in the AWS account.

These resources typically exist because your organization already installed Datadog integrations such as:

* Datadog AWS Integration IAM Role
* Datadog AWS Integration Policy
* Opsgenie API Key stored in Secrets Manager
* Datadog API Keys stored in Secrets Manager

Many organizations' Terraform pipelines need to **read** these resources in order to:

* Validate IAM roles/policies
* Fetch keys needed by other modules
* Run monitoring integration modules

### Why your module needs to know them

When Terraform runs inside GitHub Actions:

* It uses the OIDC IAM role you created
* It needs permission to **read** the Datadog/Opsgenie resources
* So your module optionally appends these resources’ ARNs to the IAM role policy

This happens only if:

```hcl
enable_datadog_permissions = true
```

If disabled:

* No Datadog or Opsgenie resources are looked up
* The GitHub IAM role stays clean
* The module behaves as a generic bootstrap module

---

## License

Licensed under the Apache License. See `LICENSE` for details.




