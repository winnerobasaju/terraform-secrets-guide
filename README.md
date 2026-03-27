# **Terraform Advanced Secrets Management Guide**

Managing secrets in Terraform safely is critical. Mismanaged secrets can leak credentials, tokens, and sensitive configuration into version control, CI/CD pipelines, or state files. This guide walks you through best practices across multiple cloud environments, with examples for **AWS** and **HashiCorp Vault**.

---

## **1. The Three Secret Leak Paths and How to Close Them**

Secrets in Terraform typically leak in three ways:

| Leak Path                            | How It Happens                                                 | How to Close It                                                                                  |
| ------------------------------------ | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **Hardcoded in `.tf` files**         | Writing passwords, API keys, or tokens directly in resources.  | Never hardcode secrets. Use external secret stores or environment variables.                     |
| **Variable default values**          | Default values are stored in `.tf` files and committed to Git. | Remove defaults for sensitive variables. Prompt users or fetch at runtime.                       |
| **Plaintext in `terraform.tfstate`** | Terraform stores sensitive resource attributes in state files. | Use remote backends with encryption and restricted access. Mark variables as `sensitive = true`. |

---

## **2. AWS Secrets Manager Integration Pattern**

**Step 1:** Store secrets manually in AWS Secrets Manager:

```bash
aws secretsmanager create-secret \
  --name "prod/db/credentials" \
  --secret-string '{"username":"dbadmin","password":"secure-password"}'
```

**Step 2:** Fetch secrets in Terraform at apply time:

```hcl
data "aws_secretsmanager_secret" "db_credentials" {
  name = "prod/db/credentials"
}

data "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = data.aws_secretsmanager_secret.db_credentials.id
}

locals {
  db_credentials = jsondecode(
    data.aws_secretsmanager_secret_version.db_credentials.secret_string
  )
}
```

**Step 3:** Use secrets in resources:

```hcl
resource "aws_db_instance" "example" {
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.micro"
  db_name        = "appdb"

  username = local.db_credentials["username"]
  password = local.db_credentials["password"]

  allocated_storage   = 10
  skip_final_snapshot = true
}
```

* Secrets are fetched at runtime, never stored in `.tf` files.

---

## **3. HashiCorp Vault Integration Pattern**

For teams using Vault:

**Step 1:** Configure Vault provider:

```hcl
provider "vault" {
  address = "https://vault.example.com"
  token   = var.vault_token
}
```

**Step 2:** Read secrets from Vault:

```hcl
data "vault_generic_secret" "db_credentials" {
  path = "secret/data/prod/db"
}

locals {
  db_credentials = data.vault_generic_secret.db_credentials.data["data"]
}
```

**Step 3:** Use secrets in resources just like AWS example.

**Benefits:** Centralized secrets, audit logging, dynamic secrets, and rotation policies.

---

## **4. Environment Variables for Provider Credentials**

* Terraform providers detect credentials from environment variables.
* **AWS Example:**

```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

* **Vault Example:**

```bash
export VAULT_ADDR="https://vault.example.com"
export VAULT_TOKEN="s.YourVaultToken"
```

* In CI/CD pipelines, inject secrets securely from your platform (GitHub Actions secrets, GitLab CI/CD, etc.) rather than hardcoding.

---

## **5. State File Security Checklist**

* Use **remote backends** like S3, GCS, or Terraform Cloud.
* Enable **encryption** (AES-256 server-side for S3).
* Enable **versioning** to recover from accidental overwrites.
* Restrict access using **least-privilege IAM roles or ACLs**.
* Lock state with **DynamoDB (AWS)** or **state locking features** of the backend.

Example S3 backend with encryption:

```hcl
terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
    encrypt        = true
  }
}
```

---

## **6. Recommended `.gitignore` for Terraform Projects**

```gitignore
# Terraform directories
.terraform/
.terraform.lock.hcl

# Terraform state files
*.tfstate
*.tfstate.backup
*.tfvars

# Override files
override.tf
override.tf.json
```

---

## **7. IAM Policy for Least-Privilege State Bucket Access (AWS Example)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-terraform-state-bucket",
        "arn:aws:s3:::your-terraform-state-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/terraform-state-locks"
    }
  ]
}
```

* Grants only the **minimum access** required for Terraform to read/write state and manage locks.

---

## **8. Publishing and Sharing**

* You can **publish this guide as a GitHub repository**, including:

  * `README.md` (the guide itself)
  * Example Terraform configurations
  * `.gitignore` template
  * Optional scripts for creating secrets in AWS or Vault

* **Tips for blog posts:**

  * Include diagrams of secret flows.
  * Show how state, environment variables, and secret managers interact.
  * Add a table of **“Do’s and Don’ts”** for Terraform secrets.

---

✅ **Key Takeaways**

* Never commit secrets to Git.
* Use runtime secrets fetching (AWS Secrets Manager, Vault, etc.).
* Protect state with encryption and access controls.
* Mark sensitive variables and outputs as `sensitive = true`.
* Inject provider credentials securely via environment variables or IAM roles.
